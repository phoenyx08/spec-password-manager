<intent>I am going to create simple password manager running natively on Linux.
Human-readable name: Your Very Own Password Manager
Machine-readable name: yvopassman
Programming language: C
GUI: GTK
Distribution: .deb package installable with dpkg
Data storage format: encrypted json
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
User should be provided with documentation about how to install, use and uninstall the program.
</intent>
<instruction>
Now I want you to create a technical description for a programmer to create the password manager.
Avoid suggesting alternatives. Suggest the most appropriate solution or suggest none.
</instruction> 
