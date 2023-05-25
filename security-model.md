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

## The security workflow for CPPServer requests

Before getting into the details of the security process, let's review the general model of CPPServer alone:

![api-definition](https://github.com/cppservergit/cppserver-docs/assets/126841556/b32159dd-01ca-4c3e-bd13-de0d0c82c6e0)

The CPPServer process reads on startup a configuration file /etc/cppserver/config.json where the microservices definitions are stored, it will parse this file once and then it will start listening to requests that should map some of the services defined in config.json, in this file, each service will define its properties, like path, C++ function to execute, SQL to execute and also some security constraints, or lack of security if required! This config.json file is central to the operation of CPPServer, here is where the no-code declarative part of the CPPServer approach resides.

So let´s review what happens when a request arrives.

1. An HTTP request to execute a microservice (GET/POST) hits a CPPServer process.
2. If the session (represented by the cookie CPPSESSIONID) does not exist in SessionDB or the cookie was not present, and the microservice is secure (default) then an HTTP code 401 (invalid request, login required) will be returned to the client, the microservice will NOT be executed.
3. If the microservice is not secured (it has the "secure" attribute set to 0 in config.json) then the microservice will be executed even if there is no valid security session associated to the request, restrictions on authorized roles are ignored if the microservice is not secure, by default all microservices are secure unless explicitly defined in config.json, and a warning when starting the CPPServer process will be printed to the logs.
4. If the request passes the authentication test but the microservice has a roles restriction defined in config.json (authorized roles that can execute the microservice) then it will compare the roles associated to the security session with the list of authorized roles, a nano-seconds operation, and if the authorization test is not passed, a JSON response with status "INVALID" will be returned, and the validation fields in the response will contain information to let the client know without ambiguity that the request was rejected for lack of permissions. For this case an HTTP code 200 is returned, please refere to the [CPPServer JSON spec](https://github.com/cppservergit/cppserver-docs/blob/main/json_response_spec.pdf) for detailed information about how CPPServer responds to different situations. In general, CPPServer will always return 200, even on validation tests and errors, unless the authentication test fails, in which case it does return 401.
5. If the request passes the authentication and authorization tests, then the microservice will be executed.

![security-workflow](https://github.com/cppservergit/cppserver-docs/assets/126841556/4dc6dffa-4d4c-41aa-961a-0d900d9cfc8f)

Please note that after the security tests have been verified, there are some validations of parameters that may fail, if this is the case the service won't be executed, but these validations are not security related, that's why these are not depicted on the diagram.

## The login process

CPPServer has built-in login capabilities, but only for SQL databases (users/passwords stored in a database table), in general, it is recommended to disable login from CPPServer and leave that task to LoginServer container, this component has the same SQL login capabilities, but it also includes a configurable LDAP login. There is a login adapter to use with a PostgreSQL database, that requires the existence of a function called cpp_dblogin and stored in the public schema, LoginServer depends on this PGSQL function object, what happens inside is not relevant, except that the function must return a certain resultset with some specific columns and be certain to verify login credentials without ambiguities, this SQL function acts as an authentication backend, and makes LoginServer independent of the implementation details of an SQL-based login mechanism. The SQL login adapter will use this function, thus this piece of C++ code is indepentent of the security model implemented in the database.

![Pod deployment model](https://github.com/cppservergit/cppserver-docs/assets/126841556/a8fe208f-6faa-422c-bce5-45e7872fc1f3)

For the QuickStart tutorial we provide a DemoDB that contains the objects of SessionDB, a security scheme with some tables that represent a legacy users' security model, and the business data tables, all in a single DB for simplicity's sake, but we have to define three different database connections as environment variables (Kubernetes secrets) in any case: CPP_SESSIONDB, CPP_LOGINDB and CPP_DB1 (the business database, you can define DB2...DB10).

This is the interface definition for the login pgsql function, it must be called cpp_dblogin and it must be stored in the public schema:

```
CREATE OR REPLACE FUNCTION public.cpp_dblogin(
	_userlogin character varying,
	_userpassword character varying)
    RETURNS TABLE(email character varying, displayname character varying, rolenames character varying) 
    LANGUAGE 'plpgsql'
```

If the login was successful a resultset of 1 row is expected with 3 columns, the "rolenames" column must contain the user's roles separated by comma ",".
An empty resultset is interpreted as a failed login. If the login succeeds then this LoginServer will create a security session for the user by inserting a row in the public.cpp_session table in the session database using another database function for this purpose which is part of the CPPServer database support layer for session management, as explained before.

Here is an example of the cpp_dblogin SQL function using our security schema from DemoDB:
```
CREATE OR REPLACE FUNCTION public.cpp_dblogin(
	_userlogin character varying,
	_userpassword character varying)
    RETURNS TABLE(email character varying, displayname character varying, rolenames character varying) 
    LANGUAGE 'plpgsql'
    ROWS 1

AS $BODY$
declare q varchar;
declare roles varchar;
BEGIN
	select STRING_AGG(distinct r.rolename, ', ') into roles from security.s_user_role ur inner join security.s_user u
	on u.user_id = ur.user_id
	inner join security.s_role r on r.role_id = ur.role_id
	where u.userlogin = _userlogin;

	q := _userlogin || ':' || _userpassword;
        q := ENCODE(decode(MD5(q), 'hex'), 'base64');
	RETURN QUERY select x.email, CAST(x.fname || ' ' || x.lname as varchar) displayname, roles as rolenames from security.s_user x where
	x.userlogin = _userlogin and
	x.passwd = q
	and x.enabled = 1;
	RETURN;
END;
$BODY$;

ALTER FUNCTION public.cpp_dblogin(character varying, character varying)
    OWNER TO postgres;

GRANT EXECUTE ON FUNCTION public.cpp_dblogin(character varying, character varying) TO cppserver;

GRANT EXECUTE ON FUNCTION public.cpp_dblogin(character varying, character varying) TO postgres;

REVOKE ALL ON FUNCTION public.cpp_dblogin(character varying, character varying) FROM PUBLIC;

```
NOTE: For security reasons, the database role used to connect to LoginDB should only have permissions to execute cpp_dblogin(), nothing more, and the permission to execute this function should be assigned to this role only.

LoginServer can also use an LDAP server if properly configured in its environment variables, it does use the standard C API for LDAP to connect to OpenLDAP or ActiveDirectory.
This is a fragment of the loginserver.yaml file used to deploy it on Kubernetes, here you can see the environment variables relevant to the security tasks, including the LDAP configuration.
```
          - name: CPP_LDAP_URL
            value: "ldap://ldap.mshome.net:1389/"
          - name: CPP_LDAP_USER_DN
            value: "cn={userid},ou=users,dc=example,dc=org"
          - name: CPP_LDAP_USER_BASE
            value: "ou=users,dc=example,dc=org"
          - name: CPP_LDAP_USERGROUPS_BASE
            value: "ou=users,dc=example,dc=org"
          - name: CPP_LDAP_USER_FILTER
            value: "(userid={userid})"
          - name: CPP_LDAP_USERGROUPS_FILTER
            value: "(member={dn})"
          - name: CPP_SESSIONDB
            valueFrom:
              secretKeyRef:
                name: cpp-secret-sessiondb
                key: connstr
                optional: false
          - name: CPP_LOGINDB
            valueFrom:
              secretKeyRef:
                name: cpp-secret-logindb
                key: connstr
                optional: false
```

