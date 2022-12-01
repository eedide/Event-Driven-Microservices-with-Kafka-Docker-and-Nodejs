# Event Driven Microservices with Kafka and Docker
This is a demonstration of event driven microservices using kafka. The application consist of multiple services, and together these services provide the overall functionality of a complete signup/login/reset password system. This is a backend application developed with nodejs and several backend tools.
![system architecture 2](https://user-images.githubusercontent.com/3667737/204997563-9bb1d963-fc89-4d37-87e4-e2aa8ed06472.png)
<H2>Explaination</H2>
There are four services that together makeup the signup/login/reset-password application - Sign-up, Activate-User-Registration, Login and Reset-Password. Each of these four services writes to its own database.
<H3>Sign-up</H3>
<ul>Please following numbering on the diagram above.</ul>
<ul>(1) Signup detail is submitted to the endpoint (http://localhost:8080/api/v1/dev/signup) and user detail is saved in mysql1 (assuming mysql1 is the primary at the moment) and an activation email is sent to the supplied email address. The record saved in mysql1 is replicated in mysql2 and mysql3 databases respectively. The user detail submitted to endpoint is in this format:
  <pre><p>{</p>
  <p>"email": "testing10@firstclicklimited.com",</p>
  <p>"password": "password"</p>
  <p>}</p></pre>
</ul>
<ul>
  (2) Next a debezium source connector (which i call signupconnector in the connector config later on) installed in kafkaconnect service listens for changes (insert, update and delete) in mysql1 database and records (3) those changes in the kafka topic called: users. This process is called change data capture (CDC).
</ul>
<ul>
  (4) jdbc sink connector for login_mysql (called login-sink-connector later on) which is a consumer of the users topic takes the user's record written to users topic and sinks (5) it to the login_mysql database immediately. Now that users records from mysql1 is sank into login_mysql, the two services: sign-up and login can work with independent databases. Login service does not have to depend on mysql1 database and vice versa. Note that only selected fields of the user's table structure of mysql1 can be extracted and sank into login_mysql (all depends on the login service requirement). Another advantage of having every service to work with its own database is that the table structure of a database can be changed to fit the requirements of a particular service without affecting another service.
</ul>
<H3>Activate-User-Registration</H3>
<ul>Note that signup process is not yet completed. Signup can only be completed if everything goes well in this service. The complete process of signup is a distrubuted transaction between Sign-up and Activate-User-Registration in the sense that if the user clicks on the email link that was sent previously and the account is successfully activated, Activate-User-Registration calls back on Sign-up alerting it to change the status of the user from INACTVE to ACTIVE otherwise alerts it that activation failed in which case the status of the user in mysql1 users table is left as INACTIVE and the transaction is completed. Note that Activate-User-Registration service also alter user status in its own database (mongodb1 - assuming its the primary) accordingly</ul>
<ul>(6)User's activation link is submitted to the endpoint (http://localhost:4000/api/v1/dev/confirmuserReg?token=one-time-jwt-will-be-here) and user activation status (ACTIVE or INACTIVE) is saved in mongodb1. ACTIVE is saved if everything is well, otherwise INACTIVE is inserted</ul>
<ul>(7)Debezium source connector listening on mongodb1 to capture data changes captures the changed data and writes (8) it to mongoserver1.activate_user_reg_db.active_users kafka topic (short formed as active_users in the diagram above)</ul>
<ul>(9)Sign-up service (which is a consumer of the active_users topic) consumes the changes made to the active_users topic (this is how sign-up service knows if users activation succeeded or failed)</ul>
<ul>(10)Sign-up service adjusts or alters its database (mysql1) users table accordingly. If user's account activation succeeded, sign-up service alters it database user's status to ACTIVE otherwise status is left as INACTIVE. Now the distributed transaction between Sign-up and Activate-User-Registration is completed.</ul>
<ul>(11) Procedure (2) is repeated</ul>
<ul>(12) Procedure (3), (4) and (5) is repeated</ul>
<ul>(13)</ul>
