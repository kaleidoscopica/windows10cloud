# windows10cloud
This is a guide to creating a Windows 10 VM in a cloud environment.

It's geared to users on neopets.com, who want to safely play old, outdated technologies like Shockwave and Flash, in order to accomplish their childhood goals.

Currently it only has steps for Google Cloud Platform (GCP). 

Users must place their trust in the provided Windows 10 VM from https://archive.org/details/msedge.win10.vmware.galoget - this is not mine and I simply trust that, as the uploader claims, it is Windows' official VM before they took it down from their own site for downloading.

## Preparing to make the VM

Prereq: A Google Cloud Platform (GCP) account, with billing already set up. (Refer to Google's documentation for how to do this.)

1. Download the official Windows 10 with MS-Edge VMDK .zip file from https://archive.org/details/msedge.win10.vmware.galoget - it's almost 7GB and may take some time to download. When complete, unzip the archive to get the MSEdge-Win10-VMware-disk1.vmdk file.

2. In your GCP Console https://console.cloud.google.com, navigate to Cloud Storage. If it prompts you to enable the API first, do so, then come back. Create a bucket. Give it a unique name. Under "Choose where to store your data", select Region. I used us-east1, but you can also choose either us-west1 or us-central1, as these are currently free to store 5 GB/month (our file is over this limit, but the first 5GBs will still be free). You can leave it as Standard storage -- this will also still be free to store the first 5 GB/month. Under "Choose how to protect object data", you can deselect "Soft delete policy (For data recovery)". Create the bucket and it warns you that public access will be prevented; that's okay, so click confirm.

3. Creating the bucket will take you to it. Click "Upload Files". Upload the MSEdge-Win10-VMware-disk1.vmdk file and wait for it to complete. 

4. Navigate to Compute Engine. If it prompts you to enable the API first, do so, then come back. Then navigate tp Migrate to Virtual Machines. If a splash page comes up asking you to enable the VM Migration API, click Enable. Then navigate to Migrate to Virtual Machines again. On the right, click Targets, and scroll down to click Add a Target Project. Select your project and click Add (1).

5. Go to the Create an image page at https://console.cloud.google.com/compute/mfce/images/create. Name your image, ex 'windows-10'. Under Source Cloud Storage file, select browse, navigate to your bucket, and select the MSEdge-Win10-VMware-disk1.vmdk file. Notice how underneath there is a warning: "Migrate to Virtual Machines service account service-123456789@gcp-sa-vmmigration.iam.gserviceaccount.com needs to have the 'storage.objects.get' permission on the selected source file". Copy/paste the full name of the service account it gives you (ex: service-123456789@gcp-sa-vmmigration.iam.gserviceaccount.com).

6. Go back to your Cloud Storage bucket, and go to the Permissions tab. Click Grant Access. Under New principals, paste in the name of your service account from step #5. Under Assign Roles, give it the Cloud Storage > Storage Object Viewer role. Grant the access.

7. Go back to the Create an image page at https://console.cloud.google.com/compute/mfce/images/create. Now we will go through the rest of the steps to create the image. First, name your image, ex 'windows-10'. Under Source Cloud Storage file, select browse, navigate to your bucket, and select the MSEdge-Win10-VMware-disk1.vmdk file. I chose us-central1 for the Region. Under Target Project, you should be able to select your project if you followed step #4 correctly. Everything else can be left as defaults; click Create. Mine ran the import for ~20 minutes.

8. Now that the image is created, you should delete the MSEdge-Win10-VMware-disk1.vmdk file from your Cloud Storage bucket, to save on costs.


## Creating the Sole-Tenant Node Group

1. To run a custom Windows VM, GCP requires you to run it on a sole-tenant node. This seems to be due to Microsoft licensing terms requiring GCP to run Windows 10 on cloud hardware dedicated to you and you alone (i.e... blame Microsoft for the higher cost). First navigate to Compute Engine > Sole-tenant nodes.

3. Create a node group. Name it anything (ex. node-group-1). For Region, you can select us-central1, and keep (and note) whatever zone it populates. Click continue. Select the node template you just created. Click continue. Under Node template properties, click the node template box, and click Create node template. Name it anything (ex. node-template-1). For the node type, you can select the smallest one, c2-node-60-420. Leave SSD and GPU blank and click create to create the node template. Back on the node group, click continue. Configure autoscaling to On, and set Minimum number of nodes as 0, and maximum number of nodes as 1. Create the node group.

4. The sole-tenant node is not created yet. A new project likely does not have an appropriate CPU quota for this type of node yet, so we must request this, and wait. To trigger this, edit the node group you just created, and try to turn autoscaling to Off, with Number of nodes as 1. This action will fail if you do not have enough quota, but you will see a notification that prompts you to "Request Quota" - follow that link, and click the "..." on the right of the quota to select "Edit Quota". Follow the instructions. You will want to request at least 60 vCPUs. This should increase both C2_CPUS and CPUS_ALL_REGIONS quotas to 60. As justification, you can write something like "I want to create a sole-tenant node." The GCP team will probably approve your request, and it should be fairly quick, but you may need to wait a little bit.

5. Once approved, once again try to turn autoscaling to On, with Number of nodes as 1. This time it should succeed. This is the setting you should keep - it will make sure that a node is only created in your node pool once you try and create the Windows VM, and while it is actually running. When you shut down (or delete) the Windows VM, it will also remove the node that it was placed onto. This should save you money and is **good**.

6. **Timer starts NOW!** This is the major cost associated with this setup - it will charge you a few bucks per hour, and adds up quick! Go as fast as you can to finish the setup and accomplish your objectives. As long as you have autoscaling turned on properly for your sole-tenant node group, and make sure to shut down your VM when not in use, there are already some safeguards in place against the major costs -- but just in case, performing the full cleanup (see **Cleanup** section) will make 100% sure you don't accrue additional costs.


## Creating the Windows VM

1. In Compute Engine, navigate to Storage > Images. You should see your windows-10 image at the top of the list. Click the '...' under Actions on the right, and Create instance.

2. Name the VM; I named it 'windows-10'. I kept the Region as us-central1. You will need to select the zone that your sole-tenant node is in. You also need to select the c2-standard-4 machine type (under Compute optimized) to match the sole-tenant node. Under Advanced Options, at the very bottom, expand the Sole-tenancy dropdown - it prompts you to input a node affinity. You have to click Browse and match it with your sole-tenant node pool. (If it takes too long to display anything, click Cancel and try again and it should find it more quickly.) When done, click Create.

3. It's a bad idea to let just anyone RDP to your VM. You need to whitelist your unique IP address, so ideally your IP stays stable (most should stay fairly stable over a few days, if you are using the same computer, on the same network). To find out your IP, go to icanhazip.com and copy the result. If you lose ability to RDP to your VM on a subsequent day, check whether your IP has changed by again visiting icanhazip.com.

4. Go to VPC Network > Firewall. You need to create a firewall rule to allow your IP to access your VM. Click Create Firewall Rule, at the top. Name the rule 'allow-my-rdp' (or you can name it anything you want). Under Targets, select All instances in the network. Under Source IPv4 Ranges, paste your IP, then append /32 to it before leaving the box (ex: 1.2.3.4/32). Under Protocols and ports, make sure Specified protocols and ports is selected, and check the box for TCP, and in the TCP box, enter 3389. Leave everything else alone. Click Create.


## Accessing the VM

1. MacOS: Go to the App Store and download the Microsoft Remote Desktop app. After initial setup (you can allow microphone/camera access) click "Add PC".

   Windows: On your local Windows PC, you can use the search box on the taskbar, type Remote Desktop Connection, and then select Remote Desktop Connection. You can also download and use the Remote Desktop app, from the Microsoft Store.

3. In the GCP console, navigate back to VM instances, under Compute Engine. Note that your windows-10 VM has an external IP listed in the External IP column. Copy this value.

4. Paste the external IP into the PC Name (or Computer) field. Leave everything else as defaults and click Save (or Connect).

5. If on a Mac, you may need to double click the new box that popped up and let it start connecting.
  
6. When it prompts you for a username, enter `IEUser`. For password, enter `Passw0rd!` and connect. It warns you that the certificate cannot be verified; this is fine.


## Accomplishing your Neopets goals 

1. Once in the VM, click the "e" in the taskbar. Ignore it when it prompts you to "download Edge". Just go straight to the search bar and Google "chrome" to find, download, and install Google Chrome. (This version of Edge does not have great compatibility with viewing the Github guides linked in the next section, so it's better to use Chrome.)

2. Open Chrome and go to https://github.com/SpudMonkey7k/neopets-IE. Follow the steps in there to download and install Fiddler, Flash, Shockwave, and 3dvia. 

**Gotchas**:
- Because this version of Windows 10 is running in the cloud, Fiddler prompts you to use WinConfig to enable traffic capture. Follow its advice - click WinConfig, click Exempt All, and Save Changes.
- I did have to install Flash in Windows 7 compatibility mode, and as Administrator.
- Internet Explorer is already installed on this version of Windows 10 (you can find it via the start menu/search), so you don't need to follow the section called "Opening IE (Windows 10 and up)". **Do** go down to "Initial IE Setup" and follow those steps, though.
- If you have Neopass, logging in to Neopets first via Chrome is helpful to get your document cookie and copy that over to IE (bottom steps on https://github.com/SpudMonkey7k/neopets-IE under Neopass).
- The guide has a warning that "If IE freezes trying to load Neopets, please remove neopets from Compatibility Settings!" - I did find this to be the case for me.
- Many Neopets main pages don't load in IE, so go directly to https://neopets.com/games/classic.phtml to go to the game library. 
- The game I tested to make sure Shockwave was working was Hannah and the Pirate Caves. It initially warned me that "It appears that this game is not running at its intended location." Following others advice, I had to press and hold shift + o + k while loading. Loading the game in the lowest setting also seemed to help. 
- I haven't done much other testing yet, and I didn't try to send a score, so let me know if there's anything missing.


## Done?

1. Shutting down the VM is easy and can be done from within the OS (normal shutdown procedure). After a minute, you should see the VM has stopped, when you view it on the Compute Engine dashboard. You can also stop it from the dashboard by checking its box and hitting stop. You can start it again in the same way, but click Start/Resume.

2. As previously mentioned -- as long as you have autoscaling turned on properly for your sole-tenant node group, and make sure to shut down your VM when not in use, there are already some safeguards in place against the major costs -- but just in case, performing the full cleanup, that includes _deleting_ the VM (see **Cleanup** section) will make 100% sure you don't accrue additional costs. As is, the VM does not incur CPU/memory charges while shutdown (assuming your autoscaling also shuts down the node from the sole-tenant node group afterward), but it has a 40GB boot disk that continues accruing storage costs until you fully delete it.
   

# Cleanup

***THIS IS VERY IMPORTANT!*** Failing to clean up your resources properly could have GCP charge you hundreds of dollars over the next days/weeks!

1. On the main Compute Engine dashboard, delete your windows-10 VM. (If it's only shut down, most of the 

2. Under Sole-tenant nodes, delete your sole-tenant node-group.

3. If you haven't already done so, navigate to your Cloud Storage bucket and delete the MSEdge-Win10-VMware-disk1.vmdk file from the bucket.

4. Go to Storage > Images and delete the windows-10 image.

5. At this point, nothing should be accruing costs anymore, but you should double-check by following up in the **Billing** section both the next day, and the day after that -- sometimes costs take a day or two to post. If you still see services accruing costs, that section will help you find them and shut them down.
