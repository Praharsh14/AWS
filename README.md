# Installing specific versions of technologies on ec2 instance (Linux/Debian).
 > **Note:** Install only the packages that are required; unnecessary packages will take up unused space.  
> Some packages may already come pre-installed with your server.


1. ### Update the packages
- `sudo apt update && sudo apt upgrade -y`
2. ### Node installation
- Install NVM (Node Version Manager)
	```bash
	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
	source ~/.bashrc
		
	# Install Node.js v20.11.1
	nvm install 20.11.1
	nvm use 20.11.1
	```
3. ### Python Installation
- check all all available versions of the python3.10
- `apt list -a python3.10`
- Then install the specific version by:
- ` sudo apt install python=3.10.12`

4.  ### NGINX installation
- First, check available versions for a specific variant (let’s say nginx-full):
- `apt list -a nginx-full`
- Then install the exact version:
- `sudo apt install nginx-full=1.18.0-6ubuntu14.6`
5.  ### Git Installation
- Check available versions:
- `apt list -a git`
- Install that exact version (for e.g)
`sudo apt install git=1:2.34.1-1ubuntu1`
6. ### Java Installation
- Check available OpenJDK versions.
- ``` bash
	apt list -a openjdk-*  #This will show you whole openjdk version list
	apt list -a openjdk-11-jdk #This will show you only for openjdk-11
- Install the specific version
`sudo apt install openjdk-11-jdk=11.0.28+6-1ubuntu1~22.04.1`



 
