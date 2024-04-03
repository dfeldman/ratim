# RATIM 
Remote Attestation using TPM and Instance Metadata

## Description 
It is very common for a service (non-human user) to need to prove its identity to a remote system. 

In the cloud world, this is typically done with an Instance Identity Document. AWS, GCP and Azure all provide slightly different Instance Identity Documents that can be used to prove that the service was running on a particular instance. 

It is also ideal to prove that the instance hasn't been compromised or tampered with when it is making that call. There is a standard called TPM (Trusted Platform Module) that cryptographically stores a variety of key system measurements in a secure, isolated module away from the OS. All cloud providers now support TPM. 

The purpose of RATIM is to tie instance identities and TPM measurements together. When the system boots, it runs a RATIM boot process which gathers all the TPM measurements and instance identity information, and stores it in a file on disk with restricted access. It then stores the hash of this document in the TPM, in a PCR register that can't be tampered with. 

Later on, at any point, the system can send the RATIM document and use the TPM "quote" mechanism to cryptographically prove that the document has not been modified since boot. 

## Why not do TPM attestation and instance identity metadata verification as two separate steps? 
If TPM measurements and instance identities are not bound together in some way, an attacker could always take instance identity metadata from a "known good" instance and copy it to one with a tampered TPM, or take a TPM measurement from a "known good" instance and copy it to one with different instance identity. There would be no way to determine, remotely, that this had occurred. 

## Does RATIM provide any value on non-cloud hardware? 
Non-cloud hardware doesn't have any concept of instance identities, so there is nothing to bind the TPM measurements to. However, the goal is for RATIM to work the same everywhere, so it will produce a document in the same format, but with empty fields for the instance identity. It will still be possible to use the other fields in the RATIM document for attestation.

## Why allow only one RATIM measurement per boot?
Only one RATIM measurement is allowed per boot cycle to prevent the "evil container" attack. 

On multi-service platforms like Kubernetes, an attacker might be able to run their own container on the instance. This would allow them to get a valid RATIM document (since it would genuinely have real access to the TPM and to the instance identity metadata), but be able to use that document to prove it has the same identity as the host (which is undesirable). To prevent this, the RATIM data hash is stored in a PCR register, which cannot be modified after it is initially created. 

Having access to the RATIM document then indicates that a service is a privileged user; is running on the node with the included instance identity; and has the TPM measurements included in the document. 

On instances running a single service, the evil container attack is not possible. However, it's best to use this approach anyway for consistency. 

## What can you do with the final RATIM document? 
In the future, you will be able to exchange it for a certificate and private key that you can use to prove the identity of the instance in network connections. 

## Can you use it for full OS integrity monitoring?
RATIM doesn't try to be a full implementation of integrity monitoring. But because the RATIM document includes important fields from the TPM, it is possible to link it to an integrity monitoring system. 

## How does this affect SPIFFE? 
Right now, SPIFFE servers always have to have a high-availability, secure database. The reason is that they have to keep track of every instance that has attested, in order to prevent the "evil container" attack. Running a potentially large, high-availability, and secure database is expensive and time-consuming. 

With RATIM, it will be possible to have completely stateless SPIFFE implementations. The client itself stores all the needed metadata, and proves that it is correct using the TPM. The server just has to validate this data and issue the right certificate. No runtime state is needed on the server side at all. Of course, the server still needs configuration information, and the root signing keys to be able to sign certificates. 

Hopefully, this will lead to simpler, more scalable, and more reliable SPIFFE implementations. 

## Do all platforms support TPM? 
Nearly all platforms now support TPM, because it is required for Windows. That includes all major cloud providers, all major hypervisors, and all new x86 hardware. 

