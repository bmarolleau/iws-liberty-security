# iws-tai-security
## Secure Web Services: IWS (Liberty) & Trusted Application Interceptor (TAI)
Link to the full presentation "[IWS, TAI, Ngninx](https://ibm.box.com/s/dnv8rhh2ikim70t69kcjf9qblehblc0u)" 

This sample Java class provides the following functions:
- JWT validation based on the JwtConsumer API (Liberty JWT feature) - the JWT signature will be checked, and the claims (exp, iss, aud) will be validated against the expected values;
- Assign the identity by retrieving the subject claim;

### How it works

When the Liberty server is launched, it will read the jwtConsumer configuration named myJWTConsumer and make it available. The TAI class will also be initialized, but in this sample nothing is done during the TAI initialization process.
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

![IWS TAI using class ](./images/iws-security-tai.jpg)

### Tests - using Liberty JWTBuilder

![IWS TAI Maven jwtbuilder ](./images/iws-security-jwtbuilder-jwtconsumer.jpg)
