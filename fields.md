#Field level encryption design
##Abstract
Service Anywhere (SAW) is a ticketing system, such that each ticket display includes a form with multiple fields 
like description, priority, etc. Administrators will be able to define encryption domains (domain for HR, domain for finance, etc.) 
and associate users to these domains. Upon association, they also create a passcode to these users. Then, the 
administrators will be able to define certain text fields as accessible, for both write and read actions, only for a 
certain domain. Since the data in these fields is sensitive, it should be saved to the server encrypted and only 
the client side, given the user password, can decrypt the data.

##High level design

We will be using client-side encryption, so no sensitive decrypted data is ever stored on the server side. 
That includes passwords and encryption keys as well as the data itself.
Each domain will have it's own unique symmetric encryption key that will be replicated to each domain user's accounts 
and encrypted with the user's own password. 
Granting a user access to specific domain involves a two step operation when administrator user copies domain key to the
user's account encrypting it with a temporary passcode that is being passed to the user and then the user confirms the 
join of the domain by entering her password and received passcode.

**Alternative approach**
Store the domain keys centrally encrypted with domain passkey and have a user enter the passkey when she encounters an
encrypted field in the ticket form.


##Data modeling by example
###ITicketFields   
  
```JavaScript
[{
	name: "title",
	title: "Title",
	type: "text",
	domain: null
},{
	name: "priority",
	title: "Priority",
	type: "select",
	domain: null
},{
	name: "description",
	title: "Ticket Description",
	type: "textarea",
	domain: null
},{
	name: "sensitive1",
	title: "Sensitive finance info",
	type: "text",
	domain: "finance"
},{
	name: "sensitive2",
	title: "Sensitive HR info",
	type: "text",
	domain: "HR"
}];
```

###ITicket

```JavaScript
{
	id: "248d40a2-a31d-11e5-b946-33ec746d333f",
	title: "Sample ticket",
	priority: "Blocker",
	description: "Long description text",
	// Encrypted with finance domain key:
	sensitive1: "U2FsdGVkX18XVlhMqx8ZvJE+HkbE5SXLL60kgPiyf9hky/tdXzHd/6iVVtWxy5cr"
	// Encrypted with HR domain key:
	sensitive2: "U2FsdGVkX1/b+NF5foQMyYQByPLHo3ITga7wm9HrovE9jDPTr5T/K2OYRnpWU9bD"
}
```

###IDomain

```JavaScript
{
	name: "finance",
	// Encrypted with admin domain key:
	key: "U2FsdGVkX1/u9mvCKjatOeWMdld81YDrRXR9vbEN1Bj5CV6TI6u7A94TebolvFUL"
}
```

###IUser

```JavaScript
{
	username: "vpoupkine",
	// Password hash with username salt: 
	// CryptoJS.PBKDF2(password, user.username, { keySize: 256/32 }).toString();
	password: "70a38b7483a578292a525a84b0a7e85b0583b92a8b77de0988ad006a23e01f52",
	domains: [{
		name: "finance",
		// Encrypted with user password:
		key: "U2FsdGVkX1/u9mvCKjatOeWMdld81YDrRXR9vbEN1Bj5CV6TI6u7A94TebolvFUL",
		passcodeRequired: false
	},{
		name: "hr",
		// Encrypted with user password:
		key: "U2FsdGVkX1/u9mvCKjatOeWMdld81YDrRXR9vbEN1Bj5CV6TI6u7A94TebolvFUL",
		passcodeRequired: true
	}]
}
```

##Admin Scenarios

###Creating the first admin user

When the server DB does not contain any users and domains, new admin user will be created upon the first system access.
The following steps will be executed on the client in case no users/domains were defined yet:

1. Generate admin domain encryption key
2. Obtain first admin username and password from user
3. Generate password hash for the first admin user
4. Encrypt the `admin` domain key with first admin user password hash as encryption key
5. Create `admin` domain with the key set to result of the previous operation
6. Submit first admin user credentials with domain set to "admin" and the `admin` domain key to the server.
7. The server should validate that no users/domains exist to accept the first user creation request.


###Creating a new domain

1. Obtain new domain name
2. Generate new domain key
3. Encrypt the new domain key with `admin` domain key
4. Store new domain in backend

###Assigning a user to a specific domain

####Admin part
1. Generate temporary passcode
2. Decrypt `admin` domain key with admin user password
3. Decrypt target domain with the `admin` domain key
4. Encrypt target domain with the temporary passcode
5. Store assigned domain to user record
6. Send temporary passcode notification to the user

####User part
1. Receive a temporary passcode notification from admin
2. Decrypt assigned domain key with the passcode
3. Encrypt assigned domain key with user's password
4. Submit password-encrypted domain key back to the server
5. Throw away the passcode :)


##User scenarios

###Login

1. Submit username and password
2. Create password hash and verify against backend
3. Obtain user bearer key and user info from the server
4. Decrypt user assigned domain keys with plain text password
5. Cache bearer key and decrypted domain keys in sessionStorage or cookie with httpOnly option or even just cache in an angular service 
but that would require entering password upon any page refresh. Need to evaluate the real risks here.

###Logout

1. Clean the bearer key and domain keys from cache
2. Invalidate views

###Change password

1. Submit new password
2. Encrypt cached domain keys with new password
3. Create new password hash
4. Submit new password hash and newly encrypted domain key to the server

###Get ticket information

1. Request ticket fields list from server (should cache that)
2. *Server Side* filter ticket fields by user domain - return only public fields and fields accessible to given domain
3. Request ticket from server by id
4. *Server Side* return only authorized fields values
5. Decrypt encrypted fields with domain key:
 
		ticketFields.forEach(function(field){
			// Do not check the domain value assuming 
			// server already has filtered out unauthorized domains
			if (field.domain) { 
				ticket[field.name] = AuthenticationService.decrypt(ticket[field.name], field.domain);
			}
		});
6. Render ticket form

**Alternative approach**

1. Upon user login obtain only the bearer key
2. Render ticket form using specially crafted directive to hide contents of encrypted fields
3. Upon user encountering an encrypted field ask the user for field's domain passcode and show decrypted field contents

###Save ticket

1. Encrypt non-public ticket fields with appropriate domain keys
2. Submit ticket fields to the server


##Potential problems and vulnerabilities

1. Caching decrypted domain key is a potential vulnerability. Not caching the key will lead to poor UX as 
the user will be required to enter her password on every occasion.
2. Should a domain key leak to unwanted party, besides re-encrypting all domain-encrypted values we will need to reset all 
user passwords or issue temporary passcodes to every user in domain in order to change the domain key system-wide.


##Implementation details

###Server side API
####TicketMeta
Provides ITicketFields by ticket id, ticket category or some other properties identifying a ticket or a group of tickets.
####TicketData
Provides tickets list and ticket details (`ITicket`)

###Client side components

####AuthenticationService
#####Login
`AuthenticationService.login(username, password) : Promise(IUser)`

Authenticate user by username/password pair and cache credentials and domain encryption key

| Name | Type | Description |
|---|---|---|
| username | string | Username |
| password | string | Plain text user password |
| `return` | Promise | Resolved to [`IUser`](#iuser) sans-password |

#####Logout

`AuthenticationService.logout()`

Clear user data and credentials/keys from the cache

#####Encrypt

`AuthenticationService.encrypt(data, domain)`

Encrypt the provided plain text data with specified domain key.
If the domain key is not available in cache assume user session expired and re-login the user: 
throw `notLoggedIn` exception, broadcast `login` event, pop up a login dialog, etc.

| Name | Type | Description |
|---|---|---|
| data | string | Plain text data |
| domain | string | Domain to encrypt the data for |
| `return` | string | Data encrypted with user's domain key |

#####Decrypt

`AuthenticationService.decrypt(data, domain)`

Decrypt the provided plain text data with specified domain key.
If the domain key is not available in cache assume user session expired and re-login the user: 
throw `notLoggedIn` exception, broadcast `login` event, pop up a login dialog, etc.

| Name | Type | Description |
|---|---|---|
| data | string | Data encrypted with user's domain key |
| domain | string | Domain to decrypt the data for |
| `return` | string | Decrypted plain text data |

