======================
Autograph Architecture
======================

.. sectnum::
.. contents:: Table of Contents

Overview
--------

Autograph exists to consolidate the storage and operations of cryptographic key
inside a high-security service, and avoid having individual services implement
and operate their own crypto.

Autograph is a web service designed to issue digital signatures for files, data
or hashes submitted to its API endpoints. It supports a variety of signature
formats, called signers, and exposes them through a JSON API that requires strong
authentication of incoming requests.

Clients connect to the autograph service via HTTP and authenticate using a
Hawk Authorization header. The signing request contains the data to be signed,
the signature format requested and eventually the name of the signer (or one
is picked automatically based on the client authorization).

Autograph first verifies the authorization against its own configuration, and
makes sure the request is not a replay by checking its nonce database. The
request is then passed over to a signer that implements a given type, such as
"Content Signature" or "XPI". The signer uses its private key to calculate the
signature, which is returned to the client in the HTTP response.

Some signers use HSMs to store cryptographic keys, in which cases the signer
asks the HSM to run the crypto operation instead of doing it itself.

 ::

    Autograph                        +---------------------------------------------------------+
    Architecture                     |                       +--------+     +----+   +-------+ |
                                     |                    +-->Signer A+----->SIGN<---+Private| |
                                     |                    |  +--------+     +----+   |Key    | |
    +-----------+                    |                    |  +--------+              +-------+ |
    |           +------data--------->+  +--------------+  |  |Signer B|                        |
    |  client   |   HTTP POST        +-->authentication+--+  +--------+                        |
    |           <----signature-------+  +--------------+     +--------+    +---+  +-------+    |
    +-----------+                    |      |      |         |Signer C|---->HSM--->PrivKey|    |
                                     |      |      |         +--------+    +---+  +-------+    |
                                     | +----v--+ +-v---+     +--------+                        |
                                     | |authori| |nonce|     |Signer D|                        |
                                     | |zations| |check|     +--------+                        |
                                     | +-------+ +-----+                                       |
                                     +---------------------------------------------------------+

Autograph is stateless, but some signers rely on a Postgres database to
manage their configurations.

Implementation
--------------

Autograph is written in Go because it is a clean and safe language, its crypto
standard library is top-notch, and we have in-house expertise.

Autograph exposes three functional routes (**/sign/file**, **/sign/data** and
**/sign/hash**), plus a handful of technical ones. When a request arrives, it
is processed by a gorilla/mux router and sent to the signature handler.

Due to the similarity of signing files, data and hashes, the same handler takes
care of processing all requests. The handler verifies the hawk authentication
token, passes the signing request to the identified signer, and returns the
encoded signature back to the client.

The authentication/authorization model is probably the most complex part of the
autograph core. Clients are required to provide a Hawk authorization with payload
signature issued by a user trusted by autograph. The `authorization` section of
the autograph.yaml configuration lists permitted users, along with the signers
each is allowed to use.

When processing a new request, autograph first verifies the validity of the
authorization header against the users it knows about, then checks the nonce
of the header against a local cache, and finally verifies the authenticated
user is permitted to use the requested signer. Should all these steps succeed,
the signing request is passed along to a signer.

Signers are separate Go packages defined under the
`github.com/mozilla-services/autograph/signer/...` package. Each signer implements a specific
type of signing, for example:

* **github.com/mozilla-services/autograph/signer/contentsignature** implements a signing
  protocol inspired by `http-miser`_ and used to sign data sent from backend
  services to Firefox user agents. The protocol is described in details in
  `Firefox Content-Signature`_. The implementation in Autograph is described in
  the `content-signature signer's README`_.


* **github.com/mozilla-services/autograph/signer/xpi** implements the PKCS7/SMIME detached
  signature protocol used to sign Firefox add-ons. The protocol is described in
  details in `Add-ons/Extension Signing`_. The implementation in Autograph is
  described in the `xpi signer's README`_.

.. _`http-miser`: https://github.com/martinthomson/http-miser

.. _`Firefox Content-Signature`: http://wiki.mozilla.org/Security/Content-Signature

.. _`content-signature signer's README`: https://github.com/mozilla-services/autograph/blob/master/signer/contentsignature/README.rst

.. _`Add-ons/Extension Signing`: https://wiki.mozilla.org/Add-ons/Extension_Signing

.. _`xpi signer's README`: https://github.com/mozilla-services/autograph/blob/master/signer/xpi/README.rst

Signers can implement three interfaces: `FileSigner`, `DataSigner` and
`HashSigner`, which correspond to the endpoints `/sign/file`, `/sign/data` and
`/sign/hash` respectively. When a signing request is received, autograph checks
if the requested signer implements the interface for the type of signature
requested. If the requested signer doesn't support a given mode (eg. the xpi
signer doesn't support the HashSigner interface), then an error is returned to
the client.

Threat Model
------------

* An attacker who gains access to Hawk credentials and is in a position to make
  requests to the autograph service can sign arbitrary data. Autograph makes no
  attempt at verifying the trustworthiness of the data being signed. This
  responsibility falls into the relying application.

* An attacker who is in position to compromise the physical infrastructure where
  Autograph is operated can dump key material from memory. Signers that utilize
  an HSM are protected against this attack.

* An attacker who gains access to encrypted configurations and is in a position
  to access the decryption service (which uses Sops) can steal key material without
  having to compromise the autograph service. Again, signers that use an HSM are
  protected.
