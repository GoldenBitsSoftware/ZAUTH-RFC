
# ZEBRA - Zephyr Bluetooth Authentication DRAFT

Bluetooth has become the defacto standard for connecting devices over short distances.  Everything from 
clocks, speakers, cars, to small low power IoT devices using Bluetooth Low Energy (BLE).  BLE 
is by far the most popular choice for configuring and managing IoT devices.  Bluetooth’s 
popularity has also made it an attractive hacking target.

ZEBRA is a proposal to add secure Bluetooth authentication in an easy to use manner for developers. 

### Problem description


While the Bluetooth protocol has become more secure and robust, one of the biggest challenges is 
authenticating a Bluetooth connection.  For consumer products, authenticating a Bluetooth 
enabled device using a pairing code is sufficient in most use cases.  For commercial IoT 
applications, such as building lighting controls, using a pairing code is very insecure and most 
IoT devices do not have a screen or the expectation of a human in the loop.  

“Authentication” in this context means securely verifying the identity of the connecting device, be 
it a mobile application or another IoT device.  The design of Bluetooth is intended to allow 
interoperability between devices, however for many devices this is unacceptable.  Should a 
Bluetooth enabled glucose monitoring system allow any mobile application to connect? 
No. How does a low-cost, not Internet connect, lighting control device authenticate a mobile 
 application connecting via Bluetooth?  


### Proposed change

The goal of ZEBRA (Zephyr Bluetooth Authenticaiton), is to provide a simple, open source
mechanism to add Bluetooth authentication for Zephyr based devices using the Zephyr 
KConfig system.  Ease of use is critical. The developer can select an authentication method and the Bluetooth layer 
(GATT or L2CAP), the necessary code is then included into the project. 

Initially two authentication methods are proposed, DTLS and Challenge-Response.  Additional
authentication methods can be added including custom authentication.  For example, a RSA hardware 
token based authentication method.  The authentication is performed at the Bluetooth application 
level versus the stack itself.  This has both advantages and disadvantages, discussed later in this proposal.

## Detailed RFC


The ZEBRA design consists of an API layer, KConfig interface, and the underlying authentication methods.  
>An initial implementation can be found here: https://github.com/GoldenBitsSoftware/zephyr    see: `feature/ble_authentication` branch.  BLE Authentication source files are located here:  subsystem/bluetooth/services/auth_*.c.  Sample source files are located here: samples/bluetooth/authenticated_connection


##### Authentication API

The Authentication API is designed to abstract away the authentication method and BLE interface 
(GATT or L2CAP).  The calling application configures the method and BLE interface, 
starts the authentication process and monitors results via a status callback.  The API is also 
designed to handle multiple concurrent authentication processes, for example If device is acting 
as a Central and Peripheral.  An example of the API used is shown in the following code snippet.

``` 
static authenticate_conn central_auth_conn;

void auth_status(struct authenticate_conn *auth_conn, auth_status_t status, void *context)
{
    if(status == AUTH_STATUS_SUCCESSFUL) {
        printk(“Authentication Successful.\n”);
    } else {
        printk(“Authentication status: %s\n”, auth_svc_get_status_str(status));
    }
}

/* BLE connection callback */
void connected(struct bt_conn *conn, u8_t err) 
{
   /* start authentication */
   central_auth_conn.conn = conn;
   auth_svc_start(&central_auth_conn);
}


void main(void)
{
    int err = auth_svc_init(&central_auth_conn, auth_status, NULL,
                             (AUTH_CONN_CENTRAL|AUTH_CONN_DTLS_AUTH_METHOD));

    err = bt_enabled(NULL);

    while(true) {
        k_yield();
}
```

##### Authentication Bluetooth Service

A new Bluetooth service is proposed, the Authentication service, would provide GATT based authentication.  It is 
acknowledged that any official Bluetooth service must be approved by the Bluetooth SIG, ZEBRA is a step in this
 direction.  For Zephyr, the authentication code will reside in the Bluetooth subsystem along with other Bluetooth 
 services in __subsys/bluetooth/services__.  The intent is to make Bluetooth Authentication a common service, easily integrated, and readily adopted 
 by developers.   The KConfig system will provide an interface to select and configure authentication.  An 
 example menu is show in the following link.
 [KCong Menu for Authentication](https://github.com/GoldenBitsSoftware/ZEBRA-RFC/blob/master/kconfig-menu.png)
 
 It is envisioned the Authentication service would be one of several other services a peripheral would 
 advertise.  For example, a glucose monitoring system would advertise the Continuous Glucose Monitoring Service 
 (UUID 0x181F) along with the Authentication Service (proposed UUID: 0x3015).
 
 For reference, a complete list of Bluetooth services can be found here: 
 [Bluetooth GATT Services](https://www.bluetooth.com/specifications/gatt/services/)
 

### Proposed change (Detailed)


BLE Authentication is implemented by a dedicated thread performing DLTS or Challenge-Response authentication 
method.   Additional authentication methods can be implemented by creating a new thread.   After the Bluetooth 
connection is established the authentication thread is started, status messages are sent to the main thread 
via a callback. The thread  terminates after authentication has successfully completed, authentication failed, 
or some other error  occurred.  The diagram in the following link shows the main components of the 
authentication service.  [Authentication Components](https://github.com/GoldenBitsSoftware/ZEBRA-RFC/blob/master/auth-block_diagram.png)

In the [Authentication Components](https://github.com/GoldenBitsSoftware/ZEBRA-RFC/blob/master/auth-block_diagram.png) 
diagram, authentication is shown with both sides (Central and Peripheral) executing the same 
code base, however this is not necessary.   A Zephyr BLE peripheral can perform a DTLS or Challenge-Response 
authentication method with a mobile phone.  The mobile phone would need to support DTLS, the same is true for
 any authentication method, both sides must understand the protocol details.
 
 #### Authentication Methods
 Two authentication methods are proposed, DTLS and simple Challenge-Response.  However, the 
 authentication architecture can support additional authentication methods in the future.
 
 * DTLS. The TLS protocol is the gold standard of authentication and securing network communications.  DTLS is 
 part of the TLS protocol, but designed for IP datagrams which are lighter weight and ideal for resource constrained 
 devices.  Identities are verified using X.509 certificates and trusted root certificates.  The DTLS handshake 
 steps are used for BLE authentication, a successful handshake means each side of the BLE connection has been 
 properly authenticated.  A result of the DTLS handshake steps is a shared secret key which is then used to 
 encrypted further communications.  For the BLE authentication this key is not used, instead the existing BLE 
 link layer encryption is used to encrypt BLE communications.  However _**Why not use this shared key for BLE 
 link layer encryption?   Why not set this key into the Zephyr Bluetooth stack after the DTLS handshake?**_ This would 
 enable different key creation possibilities with various ECC curves and RSA keys.  Future versions of ZEBRA 
 should support this capability.  The Mbed TLS stack is used for the DTLS authentication in ZEBRA.
 
 * Challenge-Response. A simple Challenge-Response authentication method is an alternative lighter weight 
 approach to authentication.  This method uses a shared key and a random nonce. Each side exchanges SHA256 
 hash of Nonce and shared key, authentication is proven by each side knowing shared key.  A Challenge-Response 
 is not as secure and DTLS, however for some applications it is sufficient.  For example, if a vendor wishes 
 to restrict certain features of an IoT device to paid applications.

The proposed authentication is done at the application layer after connecting.  This  requires the firmware 
application to ignore or reject any read/write request on other  GATT or L2CAP services until the authentication process 
has completed.  This complicates the application firmware but  does enable authentication independent of a
vendor’s Bluetooth stack. In addition, the majority of embedded engineers have no desire to modify a Bluetooth 
stack. Future ZEBRA enhancements should include a set of tools and libraries to enable developers to easily add 
signing and encryption at the application layer.

**Man In The Middle Attacks (MITM):**
The Bluetooth specification does provide for MITM protection via Secure Simple Pairing (Bluetooth Version 5, 
Vol 1, Part A, Section 5.2.3), however it requires a human to perform Secure Simple Pairing, either Numeric 
comparison or passkey.  For IoT devices, requiring human intervention is not feasible due to the large numbers
of devices installed, potential errors, and labor costs. For the DTLS  authentication, the shared key 
created during the DTLS handshake can be used to either sign and/or encrypt the application data.  For the 
Challenge-Response authentication method, the shared key along with  unique session token (possibly generated 
from the challenge nonces) would be use dot sign and/or  encrypt. Signing would be done using a HMAC of the 
message payload, encryption would be done using AES-128/256

**Reconnecting:**
 Similar to BLE link layer reconnecting after two devices are bonded, a similar mechanism is needed for an 
 application layer reconnect after secure authentcation.  Maybe ==> challege response with shared key? Conditions:
 1) If the MA (mobile app) doesn't have the key in memory it initiates a DTLS.
 2) If the MA does have a key, then a simple challenge-response with the device.  Use the key as part of a hash with a random nonce and challenge. If either the MA or device fails the challenge-response, start a DTLS handshake.
 3) If the device reboots and does not have a key, the challenge-response will fail.  Start DTLS.
 4) If the device detects a different MA is connecting, force a DTLS.  This may happen anyway when the challenge-response fails.
 5) After two days (or some other time period), the MA or device (if the device can tell time) will force a new DTLS.

### Dependencies

(_Highlight how the change may affect the rest of the project (new components,
modifications in other areas), or other teams/projects._)

**TODO!!**

### Concerns and Unresolved Questions

**DTLS Performance & Overhead:**
DTLS does add additional overhead and resource usage.  DTLS testing has shown the performance is acceptable for
most applications.  The following performance numbers were measured with two Nordic nRF52840 dev kits:
* XX seconds to perform a handshake.
* XX bytes for DLTS transferred over the radio.
* XX additional memory usage.

_(NOTE: Sample applications currently being developed will be used to provide these performance numbers)_

_Google Weave Certificate compression._  Weave is an open source code solution to compressing X.509 certificates up
to 40%.  This does increase the handshake speed and reduces the amount of resources needed.  Weave compressed 
certificates will be a developer selectable option.  
See [Google Weave Digital Certs Spec](https://openweave.io/reference/specifications/Protocol_Specification_Weave_Digital_Certificates.pdf)


## Alternatives

#### Comparison of Bluetooth Pairing and Authentication
Simply put, for commercial IoT applications pairing is not authentication. The Bluetooth version 5 specification
does define three methods of authentication (Bluetooth spec version 5, Vol 2, Part F, Section 4.2.9):  a) Numeric 
Comparison, b) Passkey, and c) Out of Band.  All of these methods require a display on the device and a 
human in the loop.  In all of these options, the user is  manually performing the authentication.  For 
commercial IoT applications, this is not an option. Imagine a commercial building where hundreds 
of light switches (the IoT devices) must be manually paired. Error prone and costly work.
 

Pairing establishes an encryption key for both parties (Central and Peripheral), which is different 
from validating the identity of the remote peer.  This is true for both Legacy and LE Secure pairing. While LE
Secure pairing uses an ECC keypair and Diffie-Hellman Elliptic Curve to protect against eavesdropping,
there is no capability to identify the remote peer.

#### GATT Authentication
GATT Authentication defined in the Bluetooth spec (Version 5, Vol 3, Part G, Section 8.1) is used to 
enable a per characteristic authentication using a shared key.  It does not authenticate a Bluetooth 
peer (Central or Peripheral).


