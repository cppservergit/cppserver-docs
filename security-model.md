# CPPServer Security Model

To invoke a microservice, which is basically sending a GET/POST request to CPPServer, the client must provide a session ID cookie with the name CPPSESSIONID, the value of this cookie has to be obtained via a previous successful login using a customer provided authentication backend. CPPServer includes a separate component, LoginServer, which serves the purpose of exposing the microservices for authentication (via DBMS or LDAP), and upon a successful login, a valid session record will be created in the SessionDB database (a PostgreSQL DB required by CPPServer) and the client will receive a response with the corresponding session ID (set-cookie HTTP header), which must be sent afterwwards by the client on every request to execute a microservice, using the cookie mentioned before, this way the session can be validated, the current user's basic information (login name, email, roles) can be retrieved in microseconds and the security enforcement layer inside CPPServer can be applied before granting execution of the request. This is the basic process of authentication and authorization.
There is a third component, CPPJob, a scheduled task or job, that runs every N minutes and deletes expired sessions according to a pre-configured timeout.

![security-model](https://github.com/cppservergit/cppserver-docs/assets/126841556/763e45a1-d217-4524-be90-75234a6ea6e1)

Regardeless of the database used for the business, the session database requires PostgreSQL, it's only a single table and a few procedures and functions stored in the public schema, designed for the specific purpose of making very fast inserts and updates of single rows, since this will happen a lot during high concurrency periods. The database does not demand lots of resources, it can be easily provisioned using a VM with 4gb of RAM and docker, the [QuickStart tutorial](https://github.com/cppservergit/cppserver-docs/blob/main/quickstart.md) shows how easy it is to automate the whole installation of the software and the creation of the database, in minutes.

## The Session database

All our containers/components are stateless, CPPServer retrieves the session data en every request by using these SessionDB's objects, so all running instances of CPPServer, LoginServer and CPPJob can access it for different purposes, like creating a new session, updating a session or deleting expired sessions, CPPServer does not require sticky-sessions on the Load Balancer, which is a lot more friendly for clusters, it was in fact designed to be executed as a container on an orchestation platform like Kubernetes, it promotes scalability by being stateless, a client can "hit" any instance/pod of CPPServer on any node of the cluster, and the security session will be there, because the session information is retrieved on-the-fly on every request, in a very fast operation (the network can affect this).

As mentioned before, in order to maintain the concept of cluster-wide security session, all containers/components involved (LoginServer, CPPServer and CPPJob) use a single table stored in a PostgreSQL database named SessionDB, specifically designed to manage a high number of concurrent inserts and updates, the CPPServer SQL security script includes the objects (functions and stored procedures) to manage it using a specific database role "cppserver" in order to provide secure/restricted access to this vital table, only CPPServer should use this database role "cppserver".

It is up to the database administrator to create a specific database role "cppserver" to access the different objects (functions and procedures), and block direct access to the cpp_session table to all users/roles, including "cppserver" role. This SessionDB database is a potential attack surface, it must be protected by all means, otherwise an attacker with enough DB privileges and knowledge of the table structure could insert a fake session record and somehow transmit the information to an attacker, who using a custom HTTPS client then could send a request with a cookie which would be validated by the security layer of CPPServer, as if a previous valid login had occured. All the security model of CPPServer depends on this SessionDB be secured by all means. If the postgres (admin role) user of SessionDB gets compromised, then CPPServer can be attacked.

All the objects were designed with performance in mind, in order to optimize the time consumed by the security checks on every request, and make it no higher than a few microseconds if the network/hardware permits it.

Summary of SessionDB objects 

|Object|Type|Purpose|
|------|----|-------|
|cpp_session|table|stores session records|
|cpp_get_timeout|function|returns the session timeout in seconds|
|cpp_session_count|function|returns resultset with number of active sessions|
|cpp_session_create|function|creates a session record and returns resultset with ID and UUID for the session (the parts of the session cookie), it will delete any previous record for the given userlogin|
|cpp_session_update|function|checks if session record exists, if so it will update the last access timestamp and return resultset with session data|
|cpp_session_delete|procedure|deletes a given session record|
|cpp_session_timeout|procedure|deletes in batch expired sessions given cpp_get_timeout() value, to be invoked by CPPJob|

This is the DDL of the SessionDB objects, please note that the PostgreSQL Admin should assign a strong password for cppserver role:
```
CREATE ROLE cppserver with LOGIN NOSUPERUSER password 'UseAStrongPassword';

CREATE UNLOGGED TABLE IF NOT EXISTS public.cpp_session
(
    session_id integer NOT NULL GENERATED ALWAYS AS IDENTITY ( INCREMENT 1 START 1 MINVALUE 1 MAXVALUE 2147483647 CACHE 1 ),
    user_login character varying(60) COLLATE pg_catalog."default" NOT NULL,
    login_time timestamp without time zone NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_access_time timestamp without time zone NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ip_addr character varying(16) COLLATE pg_catalog."default",
    mail character varying(60) COLLATE pg_catalog."default",
    uuid character varying(40) COLLATE pg_catalog."default" NOT NULL DEFAULT gen_random_uuid(),
    roles character varying(200) COLLATE pg_catalog."default",
    CONSTRAINT cpp_session_pkey PRIMARY KEY (session_id)
)

WITH (
    FILLFACTOR = 50
)
TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.cpp_session
    OWNER to postgres;

CREATE UNIQUE INDEX IF NOT EXISTS cpp_session_idx_userlogin
    ON public.cpp_session USING btree
    (user_login COLLATE pg_catalog."default" ASC NULLS LAST)
    TABLESPACE pg_default;

CREATE OR REPLACE FUNCTION public.cpp_get_timeout(
	)
    RETURNS integer
    LANGUAGE 'plpgsql'
    COST 100
    VOLATILE PARALLEL UNSAFE
AS $BODY$
declare
    q integer;
begin
    q := 5;
    return q;
end;
$BODY$;

GRANT EXECUTE ON FUNCTION public.cpp_get_timeout() TO cppserver;
GRANT EXECUTE ON FUNCTION public.cpp_get_timeout() TO postgres;
REVOKE ALL ON FUNCTION public.cpp_get_timeout() FROM PUBLIC;

ALTER FUNCTION public.cpp_get_timeout()
    OWNER TO postgres;

CREATE OR REPLACE FUNCTION public.cpp_session_count(
	)
    RETURNS TABLE(total integer) 
    LANGUAGE 'sql'
    COST 100
    STABLE SECURITY DEFINER PARALLEL UNSAFE
    ROWS 1

AS $BODY$
		select count(session_id) total from cpp_session;
$BODY$;

ALTER FUNCTION public.cpp_session_count()
    OWNER TO postgres;

GRANT EXECUTE ON FUNCTION public.cpp_session_count() TO cppserver;

GRANT EXECUTE ON FUNCTION public.cpp_session_count() TO postgres;

REVOKE ALL ON FUNCTION public.cpp_session_count() FROM PUBLIC;

CREATE OR REPLACE FUNCTION public.cpp_session_create(
	_userid character varying,
	_mail character varying,
	_ip_addr character varying,
	_roles character varying)
    RETURNS TABLE(new_id integer, uuid character varying) 
    LANGUAGE 'sql'
    COST 100
    VOLATILE SECURITY DEFINER PARALLEL UNSAFE
    ROWS 1

AS $BODY$
		delete from cpp_session where user_login = _userid;
		INSERT INTO cpp_session
		(
			user_login,
			mail,
			ip_addr,
			roles
		)
		VALUES
		(
			_userid,
			_mail,
			_ip_addr,
			_roles
		) returning session_id, uuid;
$BODY$;

ALTER FUNCTION public.cpp_session_create(character varying, character varying, character varying, character varying)
    OWNER TO postgres;

GRANT EXECUTE ON FUNCTION public.cpp_session_create(character varying, character varying, character varying, character varying) TO cppserver;

GRANT EXECUTE ON FUNCTION public.cpp_session_create(character varying, character varying, character varying, character varying) TO postgres;

REVOKE ALL ON FUNCTION public.cpp_session_create(character varying, character varying, character varying, character varying) FROM PUBLIC;


CREATE OR REPLACE FUNCTION public.cpp_session_update(
	_sessionid integer,
	_uuid character varying)
    RETURNS TABLE(user_login character varying, mail character varying, roles character varying) 
    LANGUAGE 'sql'
    COST 100
    VOLATILE SECURITY DEFINER PARALLEL UNSAFE
    ROWS 1

AS $BODY$
		delete from cpp_session where session_id = _sessionid 
		and Extract(minute from current_timestamp - last_access_time) > cpp_get_timeout()
		and uuid = _uuid;

		update cpp_session
		set last_access_time = CURRENT_TIMESTAMP
		where session_id = _sessionid and uuid = _uuid;
	
		SELECT user_login, mail, roles FROM cpp_session where session_id = _sessionid and uuid = _uuid;
$BODY$;

ALTER FUNCTION public.cpp_session_update(integer, character varying)
    OWNER TO postgres;

GRANT EXECUTE ON FUNCTION public.cpp_session_update(integer, character varying) TO cppserver;

GRANT EXECUTE ON FUNCTION public.cpp_session_update(integer, character varying) TO postgres;

REVOKE ALL ON FUNCTION public.cpp_session_update(integer, character varying) FROM PUBLIC;

CREATE OR REPLACE PROCEDURE public.cpp_session_delete(
	IN _sessionid integer)
LANGUAGE 'sql'
    SECURITY DEFINER 
AS $BODY$
		delete from cpp_session where session_id = _sessionid;
$BODY$;
ALTER PROCEDURE public.cpp_session_delete(integer)
    OWNER TO postgres;

GRANT EXECUTE ON PROCEDURE public.cpp_session_delete(integer) TO cppserver;

GRANT EXECUTE ON PROCEDURE public.cpp_session_delete(integer) TO postgres;

REVOKE ALL ON PROCEDURE public.cpp_session_delete(integer) FROM PUBLIC;

CREATE OR REPLACE PROCEDURE public.cpp_session_timeout(
	)
LANGUAGE 'sql'
    SECURITY DEFINER 
AS $BODY$
	delete from cpp_session 
	where Extract(minute from current_timestamp - last_access_time) > cpp_get_timeout()
$BODY$;
ALTER PROCEDURE public.cpp_session_timeout()
    OWNER TO postgres;

GRANT EXECUTE ON PROCEDURE public.cpp_session_timeout() TO cppserver;

GRANT EXECUTE ON PROCEDURE public.cpp_session_timeout() TO postgres;

REVOKE ALL ON PROCEDURE public.cpp_session_timeout() FROM PUBLIC;
```


