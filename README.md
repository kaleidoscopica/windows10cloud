# windows10cloud
This is a guide to creating a Windows 10 VM in a cloud environment.

**Preparing to make the VM**

Prereq: A Google Cloud Platform account, with billing already set up.

1. Download the official Windows 10 with MS-Edge VMDK .zip file from https://archive.org/details/msedge.win10.vmware.galoget - it's almost 7GB and may take some time to download. When complete, unzip the archive to get the MSEdge-Win10-VMware-disk1.vmdk file.

2. In your GCP Console https://console.cloud.google.com, navigate to Cloud Storage. Create a bucket. Give it a unique name. Under "Choose where to store your data", select Region. I used us-east1, but you can also choose either us-west1 or us-central1, as these are currently free to store 5 GB/month (our file is over this limit, but the first 5GBs will still be free). You can leave it as Standard storage -- this will also still be free to store the first 5 GB/month. Under "Choose how to protect object data", you can deselect "Soft delete policy (For data recovery)". Create the bucket and it warns you that public access will be prevented; that's okay, so click confirm.

3. Creating the bucket will take you to it. Click "Upload Files". Upload the MSEdge-Win10-VMware-disk1.vmdk file and wait for it to complete. 

4. Navigate to Compute Engine > Migrate to Virtual Machines. When a splash page comes up asking you to enable the VM Migration API, click Enable. Then navigate to Migrate to Virtual Machines again. On the right, click Targets, and scroll down to click Add a Target Project. Select your project and click Add (1).

5. Go to the Create an image page at https://console.cloud.google.com/compute/mfce/images/create. Name your image, ex 'windows-10'. Under Source Cloud Storage file, select browse, navigate to your bucket, and select the MSEdge-Win10-VMware-disk1.vmdk file. Notice how underneath there is a warning: "Migrate to Virtual Machines service account service-123456789@gcp-sa-vmmigration.iam.gserviceaccount.com needs to have the 'storage.objects.get' permission on the selected source file". Copy/paste the full name of the service account it gives you (ex: service-123456789@gcp-sa-vmmigration.iam.gserviceaccount.com).

6. Go back to your Cloud Storage bucket, and go to the Permissions tab. Click Grant Access. Under New principals, paste in the name of your service account from step #5. Under Assign Roles, give it the Cloud Storage > Storage Object Viewer role. Grant the access.

7. Go back to the Create an image page at https://console.cloud.google.com/compute/mfce/images/create. Now we will go through the rest of the steps to create the image. First, name your image, ex 'windows-10'. Under Source Cloud Storage file, select browse, navigate to your bucket, and select the MSEdge-Win10-VMware-disk1.vmdk file. I chose us-central1 for the Region. Under Target Project, you should be able to select your project if you followed step #4 correctly. Everything else can be left as defaults; click Create. Mine ran the import for ~20 minutes.

8. Now that the image is created, you should delete the MSEdge-Win10-VMware-disk1.vmdk file from your Cloud Storage bucket, to save on costs.


==Creating the Sole-Tenant Node==

1. To run a custom Windows VM, GCP requires you to run it on a sole-tenant node. This seems to be due to Microsoft licensing terms requiring GCP to run Windows 10 on cloud hardware dedicated to you and you alone (i.e... blame Microsoft for the higher cost). First navigate to Compute Engine > Sole-tenant nodes.

3. Create a node group. Name it anything (ex. node-group-1). For Region, you can select us-central1, and keep (and note) whatever zone it populates. Click continue. Select the node template you just created. Click continue. Under Node template properties, click the node template box, and click Create node template. Name it anything (ex. node-template-1). For the node type, you can select the smallest one, c2-node-60-420. Leave SSD and GPU blank and click create to create the node template. Back on the node group, click continue. Configure autoscaling to On, and set Minimum number of nodes as 0, and maximum number of nodes as 1. Create the node group.

4. The sole-tenant node is not created yet. A new project likely does not have an appropriate CPU quota for this type of node yet, so we must request this, and wait. To trigger this, edit the node group you just created, and try to turn autoscaling to Off, with Number of nodes as 1. This action will fail if you do not have enough quota, but you will see a notification that prompts you to "Request Quota" - follow that link, and click the "..." on the right of the quota to select "Edit Quota". Follow the instructions. You will want to request at least 60 vCPUs. This should increase both C2_CPUS and CPUS_ALL_REGIONS quotas to 60. As justification, you can write something like "I want to create a sole-tenant node." The GCP team will probably approve your request, and it should be fairly quick, but you may need to wait a little bit.

5. Once approved, once again try to turn autoscaling to Off, with Number of nodes as 1. This time it should succeed.


**Creating the Windows VM**

1. In Compute Engine, navigate to Storage > Images. You should see your windows-10 image at the top of the list. Click the '...' under Actions on the right, and Create instance.

2. Name the VM; I named it 'windows-10'. I kept the Region as us-central1. You will need to select the zone that your sole-tenant node is in. You also need to select the c2-standard-4 machine type to match the sole-tenant node. Under Advanced Configuration, in the Sole-tenancy dropdown, it prompts you to input a node affinity - you have to click Browse and match it with your sole-tenant node pool. When done, click Create.

3. It's a bad idea to let just anyone RDP to your VM. You need to whitelist your unique IP address, so ideally your IP stays stable (most should stay fairly stable over a few days, if you are using the same computer, on the same network). To find out your IP, go to icanhazip.com and copy the result. If you lose ability to RDP to your VM on a subsequent day, check whether your IP has changed by again visiting icanhazip.com.

4. Go to Network Security > Firewall Policies. You need to create a firewall rule to allow your IP to access your VM. Click Create Firewall Rule, at the top. Name the rule 'allow-my-rdp' (or you can name it anything you want). Under Targets, select All instances in the network. Under Source IPv4 Ranges, paste your IP, then append /32 to it before leaving the box (ex: 1.2.3.4/32). Under Protocols and ports, make sure Specified protocols and ports is selected, and check the box for TCP, and in the TCP box, enter 3389. Leave everything else alone. Click Create.


**Accessing the VM**

1. Macs: Go to the App Store and download the Microsoft Remote Desktop app.

2. After initial setup (you can allow microphone/camera access) click "Add PC".

3. In the GCP console, navigate back to VM instances, under Compute Engine. Note that your windows-10 VM has an external IP listed in the External IP column. Copy this value.

4. Paste the external IP into the PC Name field. Leave everything else as defaults and click Save.

5. Double click the new box that popped up and let it start connecting. When it prompts you for a username, enter `IEUser`. For password, enter `Passw0rd!` and connect. It warns you that the certificate cannot be verified; this is fine. 

**Cleanup**

***THIS IS VERY IMPORTANT!*** Failing to clean up your resources will charge you hundreds of dollars over the next days/weeks!

1. On the main Compute Engine dashboard, delete the your windows-10 VM.

2. Under Sole-tenant nodes, delete your sole-tenant node-group.

3. Navigate to your Cloud Storage bucket and delete the MSEdge-Win10-VMware-disk1.vmdk file from the bucket.

4. Optionally (but ideally only once you have completed your goals with Windows 10, as this will make it harder to easily spin up the environment again), go to Storage > Images and delete the windows-10 image.