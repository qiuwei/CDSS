
= Dev Workshop =
Amazon Web Services and Development Environment Setup

== Learning Goals ==
- Setting up Amazon Web Services EC2 and RDS Instances
    - SSH remote connections + public-private key security
    - Snapshot management
    - Elastic compute scales + hourly billing
    - Setting up Linux server dev environment from scratch / base image
- Running batch queries on Linux server
    - Process management to allow for multiple parallel queries or allow process to keep running in background while logged off
    - (nohup command &> logFile &)

== Preconditions ==
- Account on Amazon Web Services (use shared lab account via LastPass or you can setup a free one)

== Workshop Steps ==
- Login to the AWS Management Console and then find the respective service section (e.g., EC2 or RDS)

=== Version A - Starting from blank EC2 Linux Server and Connecting to a separate RDS (Database) Server ===
(This is worth going through to learn how the different parts connect, but many will find it more convenient to just 
load up an EC2 Linux instance that already has a database locally installed. Skip to Version B in that case.)

- Startup / Restore an RDS (Relational Database Server) Instance
	Under Snapshots section, find a copy of the database server you want to clone
		(e.g., healthrex-db-snapshot or stride-inpatient-2008-2017)
		Note you will need to know the database username and password to subsequently connect 
		(Ask someone or check shared LastPass account)
	- Actions > Restore Snapshot
		Most default options are fine
		- DB Instance Class
			Specify how "big" a server you want. For testing, micro instance is good enough
		- DB Instance Identifier
			Specify a unique name so you will be able to track the instance you created
		- Availability Zone
			Specific choice is not as important as being consistent 
			(Keep all your spawned servers in the same area to minimize network latency and security risks). 
			I've usually been using "us-east-1c."
		- Auto Minor Version Upgrade
			No, don't bother if just doing some quick testing. 
			Yes, if planning to keep a server on a for a while 
			(Which you probably shouldn't be, since you're paying by the hour).

- Startup an EC2 (Elastic Compute) Instance
	These are general purpose (Linux) servers that can basically do whatever you want them to 
	(Including running relational databases, if you don't need the convenience of the RDS setup).
	
	- Launch Instance
		- Pick the Amazon Machine Image to start from
			Server (AMI) Image Snapshots and Restore is also possible here if you did a lot of custom configuration, 
			but here we want to illustrate how to get going from a "blank" starter image 
			(i.e., first option: Amazon Linux AMI)
		- Choose an Instance Type
			For testing purposes, a micro instance should again be good enough. 
			If subsequently going to be doing some heavy computing, then pick a server with more CPUs and more RAM.
		- Configure Instance Details
			Most defaults are fine.
			- Subnet / Availability Zone
				As above, pick a consistent zone so all of your servers spawn in the same place
		- Add Storage
			Defaults to 8GB local SSD storage space. 
			Should be plenty for most development, but may need more if transferring large data files through.
		- Add Tags
			Primarily just to add a "Name" tag so you can keep track of your server(s) and what you wanted to do with them
		- Configure Security Group
			Allows for more fine tuned control of which ports each server will accept communication from 
			(e.g., SSH terminal + PostgreSQL databases + Apache web servers, etc.). 
			For basic testing purposes, should be good enough to use the existing Default security group.
		- Public-Private Key Authentication
			Rather than passwords, the default connection authentication is based on public-private key pairs.
			You can just create your own private key file here if you have not done so before. Remember where you put it.
		- SSH Connection
			Under the EC2 View Instances section, find your server and can click on "Connect" for tips.
			- Your private key file must NOT be readable by "everyone" or any "group" (indicates an unsecure file).
			  This can be modifed under file properties "Get Info" or running chmod (e.g., chmod 400 privateKeyFile.pem).
			- Use an SSH client to connect to the server (likely built in as a command-line executable)
				`ssh -i "/path/to/privateKeyFile.pem" ec2-user@ec2-XX-XX-XX-XX.compute-1.amazonaws.com`

				Note ec2-user is the default superuser name, but the address of the server will be different for you. 
				You should get a warning about the authenticity of the server/host not being established. 
				This makes sense since you've never connected to this server before, and you could be the 
				subject of a "man-in-the-middle" attack. Just agree to the connection for test/dev purposes, 
				but for real applications with security risks, you should authenticate the server fingerprint
				by other channels.
		- Install Libraries and Dependencies / Package Managers
			(The `CDSS/setup/setup.sh` script is intended to facilitate many of the subsequent steps, 
			but makes some assumptions about the local setup that may not work as intended.)

			Default server image has Python and some other utilities already installed, 
			but otherwise you'll be in an empty /home/ec2-user directory. 
			Highly advisable to use a package manager to take care of installing libraries and dependencies, 
			otherwise expect to end up in "dependency hell." For example, Linux servers such as here use "yum" 
			while Ubuntu servers have apt-get, Mac OS has Homebrew. 
			Windows doesn't have such a clean system at this point, so you may end up downloading 
			custom binary installers off the internet (https://www.lfd.uci.edu/~gohlke/pythonlibs/). 
			
			For simple Python packages, Python's PIP package manager works very well. 
			Beware of using the Anaconda Python distribution. If everything you need is included in Anaconda, 
			you'll be in good shape, but it may make things much more difficult if a subsequent dependency 
			(e.g., mod_wsgi web server connector) is not directly supported in the Anaconda distribution.

			On the Amazon Linux servers, we will "sudo" (tell the superuser to do) an installation of several packages.

			`sudo yum install git`			# So we can get our application source code
			`sudo yum install python27-psycopg2`	# Allows Python to talk to a PostgreSQL database
			`sudo yum install postgresql96`		# Optional: PostgreSQL binaries, including psql for connection testing

		- Download Copy of Application Code Repository
		    `git clone https://github.com/HealthRex/CDSS.git`
	    
	    - Setup PYTHONPATH so Python knows where to find Application Code
	    	`export PYTHONPATH=/home/ec2-user/CDSS:$PYTHONPATH`

	    	Or wherever you put the CDSS directory. Followed by $PYTHONPATH so if you already had anything set, 
		you'll still copy it here. 
		Better yet, add to the .bash_profile startup script so that it will run every time you connect to this server:
	    		`echo "export PYTHONPATH=/home/ec2-user/CDSS:$PYTHONPATH" >> .bash_profile`
	    	
	    	Or use a Unix text editor to add the export PYTHONPATH line
	    		(`vi /home/ec2-user/.bash_profile`)
	    
	    - Setup LocalEnv.py file so the application code knows where to find the RDS database
	    	Copy the CDSS/LocalEnv.py.template into a CDSS/LocalEnv.py configuration file.

	    	Edit the contents of the LocalEnv.py file's LOCAL_PROD_DB_PARAM to refer to the RDS database you setup.
	    	Look back to the configuration details of the RDS instance.
				LOCAL_PROD_DB_PARAM["HOST"] = 'YourDBID.cwyfvxgvic6c.us-east-1.rds.amazonaws.com'	# "Endpoint"
				LOCAL_PROD_DB_PARAM["DSN"] = "medinfo"		# "DB Name"
				LOCAL_PROD_DB_PARAM["UID"] = 'postgres'		# "Username" default master account name at this time.
				LOCAL_PROD_DB_PARAM["PWD"] = '<password>'	# Get this from someone else or LastPass


=== Version B - Restore an EC2 Linux Server with Local Database ===
If you just want to run some code against a local of a database, this will likely be a simpler, efficient approach.

- Restore EC2 (Elastic Compute) Instance
	- Pick the Amazon Machine Image to [Launch]
		Find and select the image in EC2 > Images > AMIs section you want to restore. For example:
		- stride-inpatient-2008-2014-postgresql - Has  local PostgreSQL database installed with inpatient data from 2008-2014
		- stride-inpatient-2008-2017-postgresql - Should be a superset of the above, updating dataset to 2017
	- Additional Configuration and Connection Steps
		See Version A above for the basic configuration steps, which should all basically be the same, 
		except for "Add Storage" there should be a default Root volume (100GB or larger) with a locally installed database.

	- Pre-Installed Server Setup
		Once the EC2 server is started / restored and you're able to connect to it, you should find several things there
		~/CDSS directory with code repo. But run `git pull` to make sure it's up-to-date.
		~/CDSS/LocalEnv.py should already have the connection parameters to the local database.
		~/.bash_profile should already have some auto-startup configurations (runs everytime you connect to server).

		Depending upon your needs, you may still need to install or update some additional library dependencies 
		(e.g., `sudo yum install XXX` or `sudo python -m pip install XXX`).

== Testing and Running (Batch) Processes ==
On EC2 Linux Server:

- Run a unit test to verify functional setup
	`python -m medinfo.db.test.TestDBUtil`
	(For more thorough testing, CDSS/TestCDSS.py will run all unit test modules in the medinfo directory tree.)

- Test a simple database query from the application code
	`python -m medinfo.db.DBUtil "select count(*) from clinical_item"`

- Go to CDSS/scripts/DevWorkshop and Try running an example script to process data from database
	`python ExampleQueryApp.py -c Lab -i A -p 0 -`

	Should output a bunch of data rows to the console. 
	If you don't want to see it all, you can send the output to a text file

	`python ExampleQueryApp.py -c Lab -i A -p 0 results/dataRows.tab`

- Artificially simulate a slow/long process by adding in a 0.1 second pause between each result row
	`python ExampleQueryApp.py -c Lab -i A -p 0.1 results/dataRows.tab`

	When you get bored waiting for above to finish, Ctrl+C to abort the process

- Run the process again, but do so in the background
	`nohup python ExampleQueryApp.py -c Lab -i A -p 0.1 results/dataRows.tab &> log/progress.log &`

	Above will run the process in the background (ending &) and continue even if you logoff (nohup = "no hangup"). 
	So you can start a long process and just let the server continue to work on it, 
	without requiring you to keep your (laptop) client computer logged in.
	Any error messages, progress indicators, or other text that you normally see in the console window will be
	redirected (&>) to the specified log file (log/progress.log)

- Check on the progress of the process you have running in the background
	`ps -u ec2-user -f`
		Checks which processes are running under the ec2-user, with full details. Note the Process ID (PID)
	`kill <PID>`
		If you need to kill a process that you don't want to continue anymore
	`top`
		Running monitor of all the most intensive processes running on the server
		Overall reporting can track how much total free memory (RAM) the server still has available, 
		and how much processor (CPU) is being used. Helpful when trying to gauge the bottleneck for intensive processes 
		(need more processors or need more RAM?). Note that total CPU load can be >100% for servers with multiple CPUs. 
		The whole point of using a multi-processor server is that you should run multiple simultaneous (parallel) processes
		to take advantage of extra CPUs working for you. You can't make a single process run at 200% speed with two CPUs,
		but you can break up the work into two separate tasks, and have each running at 100% on separate CPUs.
		Beware that the multiple processors are both using the same shared memory (RAM), so if you have a process
		that uses a lot of RAM, parallelizing the process will also multiply the amount of total RAM needed.
		
		"M" to sort the results by which processes are using the most memory
		"q" to quit/exit when done.
	`cat log/process.log`
		Show the output of the redirected console output from your application process
	`tail -f log/process.log`
		Show just the last few lines of the redirected console output, and continue watching it until Ctrl+C to abort.
		(Ctrl+C will abort the "tail" monitoring process, not the original application process.)

- Use a batch driver script to run multiple (parallel) processes
	`bash batchDriver.sh`

	Though bash (.sh) scripts are more common, I often prefer Python when it can do all of the above, is more flexible, 
	platform independent, and unifies the programming/scripting language used. 
	For example, rather than copy-pasting a dozen similar but different command line calls in the batchDriver.sh script, 
	I use a Python loop to effectively generate those commands in the script below and spawn them via the subprocess module.

	`python batchDriver.py`

	If you prefer serial, rather than parallel, processes for more control. 
	Remove the "nohup &" background commands from the .sh script, or change the Python subprocess.Popen to subprocess.call. 
	You may then want to run the batchDriver itself as a background process with a redirected log file:

	`nohup python batchDriver.py &> log/driver.log &`

	Additional support functionality:
		medinfo/common/support/awaitProcess.py - Wait until an existing process completes before starting another one
		medinfo/common/ProcessManager.py - Not implemented yet (5/14/2018). Intended to consolidate above support functionality.

	Large compute clusters often have their own job submission and parallelization schemes (e.g., qsub, bsub grid engines). 
	AWS for example has Elastic Map Reduce (EMR) and AWS Batch services. Depending on the scale of your needs, 
	you may want to look into such services. Otherwise, you can get a lot done cheaply by just taking advantage of 
	multiple CPU servers as above. For example, once you've got your EC2 compute setup, create a SnapShot AMImage, 
	then restore that image onto a new, bigger server with dozens more CPUs and then just run your processes on that server. 
	Don't forget we're paying for these servers by the hour, but the pricing is proportional to capacity. 
	Given that proportionality, you can pay twice as much for twice as many CPUs that will get your job done in half the time. 
	This is perfectly worth it since the amount of dollars spent is the same, but you save half your human time waiting for results.
