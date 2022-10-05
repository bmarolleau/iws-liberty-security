# iws-tai-security
## Secure Web Services: IWS (Liberty) & Trust Authentication Interceptor (TAI)
Link to the full presentation "[IWS, TAI, Ngninx](https://ibm.box.com/s/v2na4axu5hf48xvsy3jn6jrfd4bem1sv)" 

This sample Java class provides the following functions:
- JWT validation based on the JwtConsumer API (Liberty JWT feature) - the JWT signature will be checked, and the claims (exp, iss, aud) will be validated against the expected values;
- Assign the identity by retrieving the subject claim;

![IWS architecture overview ](./images/iws-ArchOverview-TAI.jpg)

### How it works

When the Liberty server is launched, it reads the jwtConsumer configuration named myJWTConsumer and makes it available. The TAI class is then also be initialized, but in this sample nothing is done during the TAI initialization process.
- **isTargetInterceptor** method: The TAI Class will only intercept HTTPS requests that contain a JWT in the Authorization Bearer header. isTargetInterceptor method returns 'true', negotiateValidateandEstablishTrust is then invoked.
- **negotiateValidateandEstablishTrust**  method:  will create an instance of a JwtConsumer based on the myJWTConsumer configuration to validate and parse JWTs. Once selected to handle the authentication, it will validate the JWT signature with the trustedAlias certificate taken from myJWTConsumer and will parse the JWT to retrieve the claims. The JwtConsumer only verifies the issuer, audience and expiry claims; to validate the other claims some lines of code need to be written in the TAI negotiateValidateandEstablishTrust method. 
- **Result:** If the JWT passes all the checks, the subject claim will be defined as the Principal identity and the request will be processed. Otherwise the request will be rejected.
**The subject claim needs to match an entry in the user registry!**
In this example, the JWT token subject claim has to match an IBM i user profile, and has to be member of the API_user IWS' role.

Please refer to this excellent [github repo](https://github.com/cicsdev/cics-java-liberty-tai-jwt) (IBM Z Lab Services Team , Montpellier France) for more information.

## Building a TAI class (jar) with Maven
Install maven on your system (yum install), git clone this repo and run:
-  `mvn clean package`

The generated class is packaged in a Jar file in the target directory.
Please adjust the pom.xml dependency file based on the IWS / WAS Liberty version you are using. 

![IWS TAI Maven build ](./images/mvn-package.jpg)
 
## Using a TAI Class with Liberty / IWS

1. Add the TAI class in IWS Configuration as shown below:
Adapt the path according to your setup:
![IWS TAI using class ](./images/iws-security-tai.jpg)

2. As is, the SimpleTAI class instanciates a JWTConsumer that requires the jwt-1.0 feature in Liberty: 
Add the jwt feature: 
`vim /www/wservice/wlp/usr/servers/wservice/server.xml`
(please adapt the server.xml path accordingly)
```xml
  <featureManager onError="WARN">
   ...
     <feature>jwt-1.0</feature>
  </featureManager>
```

3. **Specific to this example with JWT:** Add the following jwtConsumer line:
```xml
  <jwtConsumer audiences="acmeair1" id="myJWTConsumer" issuer="myibmi" trustStoreRef="KeyStoreByWebAdmin" trustedAlias="bendemo2022"/>
```
Not that the JWT Consumer needs a keystore, ensure that your jwtConsumer is pointing to an existing keystore (here: KeyStoreByWebAdmin) and certificate (here:bendemo2022), as in the example below (server.xml extract).
```xml
 <keyStore id="KeyStoreByWebAdmin" location="/QIBM/USERDATA/ICSS/CERT/Server/DEFAULT.KDB" password="{xor}MzoyPjI6LTA=" provider="IBMi5OSJSSEProvider" type="IBMi5OSKeyStore"/>
 ```
4. Edit, save **server.xml** and restart your IWS server.

Note: The jwtConsumer tag specifies which public certificate to use (trustedAlias) to validate the JWT signature. Only the public certificate is required, no need to have the private key in the keyStore. The private key is only necessary for building a JWT token by a jwtBuilder, any API Management component or Identity server).

5. Configure IWS **User Registry**

The User Registry will be used by TAI , **negotiateValidateandEstablishTrust** method.
In this example, IBM i user (profile) registry, with roles granting access to a particular web service:
![IWS TAI registry security ](./images/iws-security-registry.jpg)

6. Web Service level setting:

![IWS TAI ws security ](./images/iws-security-service-properties.png)
 
## Tests - using Liberty JWTBuilder

### Install a JWT Builder with Liberty

If you don't have a JWT generator, you can use the Liberty server to do it for you.
Simply add the following jwtBuilder tag in the server.xml configuration file:

`vim /www/wservice/wlp/usr/servers/wservice/server.xml`

```xml
  <jwtBuilder audiences="acmeair1" id="myJWTBuilder" issuer="myibmi" keyAlias="bendemo2022" keyStoreRef="KeyStoreByWebAdmin"/>
```
![IWS TAI jwtbuilder ](./images/iws-security-jwtbuilder-jwtconsumer.jpg)

- Note that here, the jwt builder that issues the JWT token use the same keystore than the TAI jwtConsumer. 
- The jwtBuilder and Consumer tags specifies the values that are expected for different claims; update the audiences and issuer values to match the JWT generator configuration. 

### Use jwtBuilder to generate a JWT Token for a user

The JWT feature exposes JWT builders with a REST API. A token can be retrieved by sending the HTTPS request: ` GET https://<hostname>:<httpsPort>/jwt/ibm/api/myJWTBuilder/token ` 
where myJWTBuilder is the id used by the configuration, hostname and httpsPort are the hostname and port of your IWS Server.
1. Use a Web Browser or curl to generate a token. This JWT Token is issued for the authenticated (HTTP Basic) user (Subject claim).
![IWS TAI jwtbuilder test ](./images/jwtbuilder-browser.png)

2. Check the generated token : https://jwt.io/

![IWS TAI jwt.io test ](./images/jwt.io.png)

## Testing IWS + TAI with jwt tokens
- public/private keys (cert) in the identity server (can be IBM i / IWS, as in the example, but can also be a third party tier like APIm or any trusted issuer)
- public key on IWS in the IFS, and used by jwtConsumer to validate and extract identity.

### No credentials: 401 Unauthorized
```bash
curl -X GET -k -i 'https://10.7.19.71:10443/web/services/GetUserInfo/uid0@email.com' 
```
```console
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="defaultRealm"
Content-Language: en-US
Content-Length: 0
Date: Tue, 12 Apr 2022 12:28:00 GMT
```


### Valid Bearer jwt token: HTTP 200 OK 
- Request is intercepted by TAI, valid token, valid subject, granted in the registry.
```bash
curl -X GET -k -H 'Authorization: Bearer eyJraWQiOiI0M19ZWGxrdkpPQTBmQ1Y3ZE9tOC1jSGJpQ3FJQ3pZRTQyN0RYeGR2MHNJIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYifQ.eyJ0b2tlbl90eXBlIjoiQmVhcmVyIiwiYXVkIjoiYWNtZWFpciIsInN1YiI6ImFjbWVhaXIiLCJ1cG4iOiJhY21lYWlyIiwicmVhbG0iOiJkZWZhdWx0UmVhbG0iLCJpc3MiOiJteWlibWkiLCJleHAiOjE2NDk3NjY1MzMsImlhdCI6MTY0OTc1OTMzM30.wchCdnyJHriyCCvTQe4oTmAFkqcpHj72FjtakbAVMAD_Kj6x1ErTOZ7x0gpjfx9Wqi1WyQLEo7SsW3lLdICWzTF5-uQfe5U_ArdmaDEiteQu-mhvc0MN8lbkEBAVkvZ6lqlnzhPetVZItXzqxNPlzc63DLUrLtXRf2jm2BPAOUz2mn3VVuD-k04C8QPoniUMpHq1Vn674EX4oiGC_6kOgeaXnr0NEmYbEmanWrCSFIUVcmzTsQhJTBICeNrNKd8-4LBx5wNs7GQ8qpciCFZUKjbBcsUVJK3l341n-HAmOq1r94os4LuL0BAzKp9DPCJfXMP0aj7LR_iERiPr2CQJJg' -i 'https://10.7.19.71:10443/web/services/GetUserInfo/uid0@email.com'
```
```json
{"userInfo":{"username":"uid0@email.com","status":"password","total_miles":1025440,"miles_ytd":0,"phoneNumber":"919-123-4567","phoneNumberType":"BUSINESS","address":{"streetAddress1":"156 Main St. RPG!!","streetAddress2":"address-na","city":"Montpellier","stateProvince":"27617","country":"USA","postalCode":"27648"}}}
```

### Invalid/corrupted Bearer token : HTTP 401 Unauthorized

```console
[4/12/22 14:36:58:201 CEST] 000000b9 SystemErr R [JWTTAI] CWWKS6031E: The JSON Web Token (JWT) consumer [myJWTConsumer] cannot process the token string. CWWKS6022E: The issuer [idg] of the provided JSON web token (JWT) is not listed as a trusted issuer in the [myJWTConsumer] JWT configuration. The trusted issuers are [myibmi]. 
HTTP/1.1 401 Unauthorized
Content-Language: en-US
Content-Length: 0
Date: Tue, 12 Apr 2022 12:38:29 GMT
```

### Expired Bearer token : HTTP 401 Unauthorized

```console
[4/12/22 14:39:35:995 CEST] 000000b0 SystemErr R [JWTTAI] CWWKS6031E: The JSON Web Token (JWT) consumer [myJWTConsumer] cannot process the token string. CWWKS6025E: The JSON Web Token (JWT) is not valid because its expiration ('exp') claim is either missing or the token expired. The expiration claim is [2022-04-12T14:28:53+0200]. The current time minus the clock skew is [2022-04-12T14:34:35+0200]. The configured clock skew is [300] seconds. 
```

### Valid Bearer token, but identity not granted for this Service (roles mapping): HTTP 403 Forbidden
- User in profile registry, can Basic auth other services, but not in api_usr role configured in IWS. 
```bash
curl -X GET -k -H 'Authorization: Bearer eyJraWQiOiI0M19ZWGxrdkpPQTBmQ1Y3ZE9tOC1jSGJpQ3FJQ3pZRTQyN0RYeGR2MHNJIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYifQ.eyJ0b2tlbl90eXBlIjoiQmVhcmVyIiwiYXVkIjoiYWNtZWFpcjEiLCJzdWIiOiJjbGl1c3IiLCJ1cG4iOiJjbGl1c3IiLCJyZWFsbSI6ImRlZmF1bHRSZWFsbSIsImlzcyI6Im15aWJtaSIsImV4cCI6MTY0OTc4MDc2MSwiaWF0IjoxNjQ5NzczNTYxfQ.dsP6x-u8BXtdKqXsZXIq0alk6C3whOnyBvsxaBNfy_7bSuLjH90S4J0Crn5p0Qb4lWuXPA3TE2WPvkx74JP4_iLLczfLoWDYSqttxdoZT4-QL8ONwSvK38oNldO0c_HiMUVxVx_bJrn7mTYNehsGHKmxp24FXVNrGXrxbvj0HS_oOORAMSYDa6P7al4rtdvdl_jtcVDPOTblJSZRv-wiv4lKNx49crpEGQGnHlmq99fJa8wXGv2zt_xLgfU6jGJY4RI3H5U4O4btZtm4PAtwyfcqJQGdgp5WNp4Q-4I5YXhQ5VvIk6lxQQ3VALdDIlgatyrcwEiqTQvZijX_Y-YsYw' -i 'https://10.7.19.71:10443/web/services/GetUserInfo/uid0@email.com'
```
```console
HTTP/1.1 403 Forbidden
Content-Type: text/html;charset=ISO-8859-1
$WSEP: 
Content-Language: en-US
Transfer-Encoding: chunked
Connection: Close
Date: Tue, 12 Apr 2022 14:28:04 GMT

Error 403: AuthorizationFailed
```

## Performance test with JMeter

Generate a jmeter script from the IWS-Generated OpenAPI swagger file.
- Download your swagger file from the IWS GUI or the IFS
![IWS Swagger Download](./images/iws-swagger.jpg)

- Install openapi-generator from the project web site https://openapi-generator.tech/docs/installation/
and run the generator 
```bash
java -jar openapi-generator-cli-6.2.0.jar generate -i swagger.json  -g jmeter
```
```console
[main] INFO  o.o.codegen.DefaultGenerator - Generating with dryRun=false
[main] INFO  o.o.codegen.DefaultGenerator - OpenAPI Generator: jmeter (client)
[main] INFO  o.o.codegen.DefaultGenerator - Generator 'jmeter' is considered stable.
[main] INFO  o.o.codegen.TemplateManager - writing file ./GetUserInfoAPIsApi.jmx
[main] INFO  o.o.codegen.TemplateManager - writing file ./GetUserInfoAPIsApi.csv
```
- Repeat the same steps for other API's/Web Services and potentially use the "Merge" feature of JMeter to merge different jmx in one jmx. Customize the generated jmeter jmx script  (datapools, etc.) & run your tests. 

- Here, we see that the HTTPS encryption and the Java Interceptor that validates a 3rd party certificate is quite transparent regarding the performance of this Web Service. 
![jmeter test](./images/iws-tai-jmeter-test.jpg)

- Average response times stays below 50ms and the throughput is not impacted here, as most of the time is efficiently spent in the RPG program DB2 for i query execution, not in the IWS/TAI processing: 
![TAI and no TAI throughput  ](./images/iws-tai-jmeter-throughput.jpg)



