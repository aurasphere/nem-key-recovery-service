# NEM Key Recovery Service

## 1. Problem
The user X has a private key associated to an account on the NEM blockchain. If the private key is lost, any XEM and mosaic associated to that account is lost forever. We want to prevent this adding the possibility for the user to retrieve his wealth without the private key. On the other hand, we still want prevent anybody else apart from the user to retrieve the key.

## 2. Proposed Solution
In order to give the user the possibility of retrieving his XEM and mosaics, we use a 1-of-2 multisig account. The user will be given one of the private keys associated to the account, meanwhile the other one will be stored on a remote database. If the first private key is lost, he can fetch the second one from the KRS, use this key to remove the lost account from the cosignatories of the multisig and add another one. Ideally, the user would then use the newly generated key and still have their recovery key on the remote database if that happens again.

This solution doesn't apply to multisig account. The reason is that a multisig account already allows the possibility of one key loss. If that's the case, the solution would be again removing the lost key account from the cosignatories and add a new cosignatory. If x (x>1) keys happens to be lost at the same time, then:
-	if the multisig account is n-of-n, it will be lost.
-	if the multisig account is n-of-m with x>m-n, it will be lost.
-	if the multisig account is n-of-m with x<=m-n, the account will still be able to sign transaction but it won't allow to add or remove cosigners.

In these cases, storing a key would add another level of security but the problem will still persist for x>2 and if we store yet another key that wouldn't help for x>3 and so on. A better solution would be for each cosignatory to use the KRS on his account so it won't lose it on the first place. For this reason, the multisig account won't be considered during the analysis of the KRS. 

## 3. Implementation
When considering the implementation of the KRS, we want to make it both reliable and simple to use.

The KRS core will consist of 3 NodeJS WebServices:
-	createKey
-	recoverKey
-	removeExistingKey

In order for the services to work, the user will generate his account private key within the scope of the KRS. The KRS will give back a key and keep the other one stored in a database. The other key may be recovered if needed or deleted in order for the user to be able to generate another account. 

The recovery of the key will be handled with the user email. Emails greatly helps in identifying the user but they have the disadvantage that they must be bounded to a NEM account in a 1-1 relationship. This usually is not a big problem though since email providers are usually free and if you are a company you will most likely have your own domain and mail server.

### 3.1 CreateKey
This service will be called when the user wants to create an account that can be recovered if his private key gets lost. Since this service uses sensitive data, it is important that the communication between client and server happens either on the same machine or through the HTTPS protocol to prevent network sniffing.

#### 3.2.1 Input
-	**email**, a valid email to be associated with the generated key. This email will be needed to recover the key. The recovered key will be sent to it.
-	**password**, a password used to encrypt with 3DES symmetric encryption the stored. This will protect the account if somehow the KRS database is compromised or if an attacker is able to get the recovery email.

#### 3.2.2 Output
-	**key**, a generated private key account. The key will be one of the 1-of-2 multisig account.
-	**multisigAccountKey**, the key of the generated 1-of-2 multisig account. This key can't be used to sign transactions.

#### 3.2.3 Business Logic
1.	If the email is already in DB, return error.
2.	Generate 3 private keys.
3.	Sends an aggregate modification transaction (no fee required for this in the technical reference?) to create a 1-of-2 multisig account.
4.	If everything goes well, stores one of the two cosignatories key in an internal database along with the email provided by the user.
5.	Returns the other account key and the multisig account key.

### 3.3 RecoverKey
This service will be called when the user wants to recover an account that he created with the createKey service.

#### 3.3.1 Input
-	**email**, a valid email registered with createKey. The recovered key will be sent to it.

#### 3.3.2 Output
-	**null**

#### 3.3.3 Business logic
1.	Query the DB for the key using the email passed as argument.
2.	If the email is not in DB, return error.
3.	Send an email with the recovered key (encrypted with 3DES). 
4.	The user will have to decrypt it using his password (it can be done either using  an utility or client-side but the password MUST NOT be sent to the server).

### 3.4 RemoveExistingKey
This service will be called when the user wants to delete data relative to an account that he created with the createKey service. The only purpose of this is to allow the manual delete of a record by the user in order to being able to use the createKey service again with the same email. 

#### 3.4.1 Input
-	**email**, a valid email registered with createKey. Used to get the record to be deleted.
-	**encryptedKey**, a valid 3DES encrypted stored key (the same that you would get back with the recoverKey service for the same email). Used to get the record to be deleted.

#### 3.4.2 Output
-	**null**

#### 3.4.3 Business logic
1.	Query the DB by email and encryptedKey. 
2.	If a record matches delete it, else return error.

<sub>Copyright (c) 2017 Donato Rimenti</sub>
