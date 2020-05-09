# How to load test websites?


<!-- Your front matter up here -->

Does your company care about their website? Do you have clients that are doing business over their websites? Are you ready for the upcoming Black Friday?

At this moment you should pay attention if your website and infrastructure behind it are ready to endure.

<!--more-->

Among all the great commercial tools lies the jewel which is free :) The solution that I will present in the next few lines can be run from your local workstation (we have decided to take it to the cloud).

I'm presenting you [Taurus](https://gettaurus.org/). We have used this tool with JMeter scripts to accomplish our goal.

Since we decided to run everything in the cloud (Azure) we needed a Linux VM that will host our project. We went with CentOS 7.

As we are using it in some other projects, we decided to go with a [Docker version](https://hub.docker.com/r/blazemeter/taurus/). So, after installing Linux, you need to install Docker and grab the Taurus image.

Since I wanted to use scripts that were uploaded locally, I had to mount a specific folder during the Docker run. At that point, I found out that I have an issue with rights on it. After a short search, I found an old friend to be a problem - SELinux. You can play with giving certain rights, but I just "kill" it.

What would be the use of a test if we couldn't create a report? Well, JMeter has a built-in function for creating very nice HTML reports, but in the local folder. Since this is run in Docker, we needed to find a way to extract the report. Yes, I know I already mounted Linux local folder, but I don't want to SSH to it to grab it and it also gets powered off after a test (automatically). This is where we decided to mount one more thing - Azure Storage File Share.

Mount Azure File Share in Linux -> <https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mount-azure-file-storage-on-linux-using-smb>

* Command example – mind the bolded variables:
sudo mount -t cifs //StorageAccountName.file.core.windows.net/FileShareName /home/taurus-admin/TaurusLoadTesting/report/ -o vers=3.0,username=StorageAccountName,password=”StorageAccountAccessKey”, dir_mode=0777,file_mode=0777,serverino

So, after creating File Share we were ready to run our Docker.

Here's the final command:

```bash
sudo docker run -it --rm -v ~/TaurusLoadTesting:/TaurusLoadTesting blazemeter/taurus /TaurusLoadTesting/profile/sanity.yml /TaurusLoadTesting/common/scenarios.yml /TaurusLoadTesting/variable/development.yml
```

Let me break this to pieces for you:

* We are running the image interactively with remove command after it finishes the job
* We are mounting local Linux folder to Docker image (previously we mounted Azure FS inside the same folder)
* Yaml files that we are calling are partially mandatory - Scenario one is. The rest is part of our playing and making things more complicated :)

If you were previously playing with JMeter this shouldn't be a big problem for you to implement. If not, feel free to contact me and I will send you the complete example template for testing.

Now you may relax and enjoy shopping in November ;)

