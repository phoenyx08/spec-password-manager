## Instruction

I want you to create a technical description for a programmer to create a password manager according to the Vision
below.
Avoid suggesing alternatives. Suggest the most appropriate solution or suggest none.

----

## Vision

I am going to create simple password manager running natively on Linux.
Human-readable name: Your Very Own Password Manager
Shorter Name for labels: YVO Password Manager
Machine-readable name: yvopassman
Version: 0.0.1
Programming language: C
GUI: GTK
Distribution: .deb package installable with dpkg
Data storage format: Encrypted JSON file (local-only, no network connections)
Encryption: AES-256 in GCM mode for authenticated encryption. Encryption key derived using PBKDF2-HMAC-SHA256 
from a user-defined master password stored via a keyring prompt at first run. Master password cached in memory for the session only.

### Project development tips

The project should be covered with autotests in order to minimize manual testing.
User should be provided with documentation about how to install, use and uninstall the program.

### User Experience

As a user a want to have a .deb package. 
After the package is installed the program should be available in the list of applications.
When I open the program window I should see the list of password titles. 
Next to each title there are buttons named `login`, `password` and `regenerate`. 
When I click `login` login is copied to clipboard. 
When I click `password` a password is copied to clipboard. 
When I click `regenerate` the password is regenerated and old password is replaced. 
Also, there is a button to add a password. 
When I click it the system should show me a form to insert a password title and login. 
When the form is filled system creates secure password automatically. 
The password manager should not make any network connections.
When the package is uninstalled the file storing the passwords should be deleted.

----
