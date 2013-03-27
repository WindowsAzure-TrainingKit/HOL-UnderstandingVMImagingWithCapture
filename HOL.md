b<a name="Title" />
# Understanding Virtual Machine Imaging with Capture #
---

<a name="Overview" />
##Overview##

In this hands-on lab you will walk through creating a customized virtual machine that is customized to be remotely manageable via PowerShell. You will then learn how to generalize it and save it is an image so that all new virtual machines provisioned from it will be remotely manageable by default.

<a name="Objectives" />
### Objectives ###

In this hands-on lab, you will learn how to:

- Create a new virtual machine
- Customize and generalize the virtual machine
- Save the image to the image library
- Provision New Virtual Machines based off of the image

<a name="Prerequisites" />
### Prerequisites ###

You must have the following items to complete this lab:

- [Windows Azure PowerShell CmdLets](http://msdn.microsoft.com/en-us/library/windowsazure/jj156055)
- A Windows Azure subscription with the Virtual Machines Preview enabled - [sign up for a free trial](http://aka.ms/WATK-FreeTrial)

>**Note:** This lab was designed to use Windows 7 Operating System.

<a name="Exercises" />
##Exercises##
The following exercises make up this hands-on lab:

1. [Creating a New Virtual Machine](#Exercise1)
2. [Customize and Generalize the Virtual Machine](#Exercise2)
3. [Saving an Image in the Image Library](#Exercise3)
4. [Provision New Virtual Machines based off of the Image](#Exercise4)


Estimated time to complete this lab: **60 minutes**

<a name="Exercise1"></a>
###Exercise 1: Creating a New Virtual Machine###

In this exercise you will walk through the steps of provisioning a new virtual machine and enabling Windows Remote Management. 

<a name="Ex1Task1" />
#### Task 1: Creating a New Virtual Machine ####

In this task you will use the Quick Create Virtual Machine creation option to provision a virtual machine to be customized.

1. Log in to the Windows Azure portal: https://manage.windowsazure.com using your Microsoft Account.

1. Click **New**, then select **Virtual Machine** | **From Gallery**.

	![Creating a Virtual Machine from Gallery](images/creating-a-virtual-machine-from-gallery.png?raw=true "Creating a Virtual Machine from Gallery")

	_Creating a Virtual Machine from Gallery_

1. In the **Virtual Machine Operation System Selection** page, click **Platform Images** and select **Windows Server 2008 R2 SP1**.

	![Virtual Machine OS Selection](images/virtual-machine-os-selection.png?raw=true "Virtual Machine OS Selection")

	_Virtual Machine OS Selection_

1. In the **Virtual Machine Configuration** page, specify _imgtest1_ for the Virtual Machine **Name** and set both password boxes. Leave the Virtual Machine **Size** with _Small_ value.

	![Virtual Machine Configuration Page](images/virtual-machine-configuration-page.png?raw=true "Virtual Machine Configuration Page")

	_Virtual Machine Configuration Page_

1. In the **Virtual Machine Mode** page, select **Standalone Virtual Machine** and set the **DNS Name**. Leave the remaining fields with their default values.
	
	![Virtual Machine Mode page](images/virtual-machine-mode-page.png?raw=true "Virtual Machine Mode page")

	_Virtual Machine Mode page_

1. Finally, leave the **Virtual Machine Options** page with the default values and finish the Virtual Machine wizard.

1. Wait until your virtual machine is created, you can check its status in the **Virtual Machines** page within Windows Azure Management portal. Then, click your Virtual Machine name to open the **Dashboard**.

	![Virtual Machine Created](images/virtual-machine-created.png?raw=true "Virtual Machine Created")

	_Virtual Machine Created_

1. In the bottom menu, click **Connect** to download an _.rdp_ file to connect to the Virtual Machine using Remote Desktop Connection.


	![Connect](images/connect.png?raw=true)

	_Connecting to the Virtual Machine_

	>**Note:** When asked for Login, write the same credentials you used while creating the Virtual Machine.

1. Do not close the connection, you will use it in the following exercise.

<a name="Exercise2"></a>
### Exercise 2: Customize and Generalize the Virtual Machine ###

In this exercise we are going to enable Windows Remote Management, configure SSL and allow remote PowerShell. 

<a name="Ex2Task1" />
#### Task 1: Configure PowerShell for Script Execution in the New Virtual Machine####

1. Under **Start** | **All Programs** | **Accessories** start **Windows PowerShell**.

1. Execute the following command in the PowerShell Console. When prompted, choose **Yes**.

	````PowerShell
	Set-ExecutionPolicy remotesigned
	````

1. Close the PowerShell console.

<a name="Ex2Task2" />
#### Task 2: Configure the HTTPS Endpoint for WinRM/PowerShell ####

In this scenario you are going to configure a script to run when the machine boots. This script will create a self-signed certificate and configure Windows Remote Management to listen on port 5986 using SSL.

1. Using Windows Explorer, create a folder in the **C:** drive called **EnablePSRemote**.

1. Go to **Folder and Search options** under **Organize** menu

1. Switch to **View** tab and_EnablePSRemote.ps1_ uncheck **Hide Extensions for Known File Types**.

	![Hide Extensions for Known File Types](images/hide-extensions-for-known-file-types.png?raw=true "Hide Extensions for Known File Types")

	_Hide Extensions for Known File Types_

1. Right click in **EnablePSRemote** window, select **New** | **Text Document** and name it _EnablePSRemote.ps1_.

1. Open the file using a text editor (e.g. _notepad_).

1. Copy and Paste the following script into **EnablePSRemote.ps1**.

	````PowerShell
	function Create-Certificate($hostname) {
		$name = new-object -com "X509Enrollment.CX500DistinguishedName.1"
		$name.Encode("CN=$hostname", 0)
	 
		$key = new-object -com "X509Enrollment.CX509PrivateKey.1"
		$key.ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
		$key.KeySpec = 1
		$key.Length = 1024
		$key.SecurityDescriptor = "D:PAI(A;;0xd01f01ff;;;SY)(A;;0xd01f01ff;;;BA)(A;;0x80120089;;;NS)"
		$key.MachineContext = 1
		$key.Create()
	 
		$serverauthoid = new-object -com "X509Enrollment.CObjectId.1"
		$serverauthoid.InitializeFromValue("1.3.6.1.5.5.7.3.1")
		$ekuoids = new-object -com "X509Enrollment.CObjectIds.1"
		$ekuoids.add($serverauthoid)
		$ekuext = new-object -com "X509Enrollment.CX509ExtensionEnhancedKeyUsage.1"
		$ekuext.InitializeEncode($ekuoids)
	 
		$cert = new-object -com "X509Enrollment.CX509CertificateRequestCertificate.1"
		$cert.InitializeFromPrivateKey(2, $key, "")
		$cert.Subject = $name
		$cert.Issuer = $cert.Subject
		$cert.NotBefore = get-date
		$cert.NotAfter = $cert.NotBefore.AddDays(180)
		$cert.X509Extensions.Add($ekuext)
		$cert.Encode()
	 
		$enrollment = new-object -com "X509Enrollment.CX509Enrollment.1"
		$enrollment.InitializeFromRequest($cert)
		$certdata = $enrollment.CreateRequest(0)
		$enrollment.InstallResponse(2, $certdata, 0, "")
	}

	$hostname = $env:COMPUTERNAME

	$output = & winrm g winrm/config/Listener?Address=*+Transport=HTTPS | Out-String

	if($lastexitcode -eq 0)
	{
		# HTTPS Listener Already Configured Return
		return
	}

	# HTTPS Listener Not Configured - Configure Now

	# Enable PS remoting
	Enable-PSRemoting -Force

	# Create the self signed certificate (Only for testing)
	Create-Certificate $hostname

	# Get the certificate thumbprint
	$certTp = get-childitem -path cert:\LocalMachine\My* -Recurse | Where { ($_.Issuer -eq "CN=" + $hostname) -and ($_.Subject -eq "CN=" + $hostname) } | Select Thumbprint
	$certTp = $certTP.Thumbprint

	# Open up 5986 for WinRM HTTPS
	& netsh advfirewall firewall delete rule name="WinRM HTTPS" dir=in protocol=TCP localport=5986
	& netsh advfirewall firewall add rule name="WinRM HTTPS" dir=in action=allow protocol=TCP localport=5986

	# Specify the newly created cert thumbprint 
	$certString = "@{Hostname=""" + $hostname + """;CertificateThumbprint=""" + $certTp + """}"

	# Create the HTTPS listener
	& WinRM create winrm/config/Listener?Address=*+Transport=HTTPS $certString

	# Enable Basic Authentication
	WinRM set winrm/config/service/auth '@{Basic="true"}'


	````

1. Save and Close the file.

<a name="Ex2Task3" />
#### Task 3: Create a batch file to use the task scheduler to launch the script on boot ####

1. In the **EnablePSRemote** folder, create a new text file named _taskscheduler.cmd_.

2. Open the file using a text editor (e.g. _notepad_).

3. Copy and Paste the following commands (replace [YOUR-PASSWORD] with the password you specified for the Virtual Machine in both locations).

	````PowerShell
	net user scheduser [YOUR-PASSWORD] /add
	net localgroup Administrators scheduser /add

	schtasks /CREATE /TN "EnablePS-Remote" /SC ONCE /SD 01/01/2020 /ST 00:00:00 /RL HIGHEST /RU scheduser /RP [YOUR-PASSWORD] /TR "powershell -file C:\EnablePSRemote\EnablePSRemote.ps1" /F 

	schtasks /RUN /TN "EnablePS-Remote"
	````

1. Save and close the file.

<a name="Ex2Task4" />
#### Task 4: Configure a Startup Task for the cmd file ####

1. Go to **Start** and click **Run** 

1. In the **Run** box write _mmc.exe_ and click **OK**.

1. In **File** | **Add/Remove Snap-In**, select **Group Policy Object** and then click **Add**. 

1. Accept the Defaults, click **Finish** and then click **OK**.

	![Group Policy Objects](images/group-policy-objects.png?raw=true "Group Policy Objects")

	_Group Policy Objects_

1. Expand **Local Computer Policy** | **Computer Configuration** | **Windows Settings** and then click **Scripts**.

	![Local Computer Policy](images/local-computer-policy.png?raw=true "Local Computer Policy")

	_Local Computer Policy_

1. Double Click **Startup** and click **Add**.

1. In the **Script Name** box and the following path: _c:\EnablePSRemote\taskscheduler.cmd_ and click **OK** twice to exit the dialog.

	![configuretaskscheduler](images/configuretaskscheduler.png?raw=true)

	_Configuring Task Scheduler_

<a name="Ex2Task5" />
#### Task 5: Generalize the Machine with SysPrep ####

In this step we will run sysprep to generalize the image. It will allow multiple virtual machines to be created having the same customized settings (remote PowerShell enabled).

1. Go to **Start** and click **Run** 

1. In the **Run** box write _c:\Windows\System32\sysprep\sysprep.exe_ and press Enter.

2. In the **System Cleanup Action** select **Generalize** checkbox and in the **Shutdown Options** select **Shutdown** and click **OK**.

	![sysprep](images/sysprep.png?raw=true "sysprep")

	_Sysprep dialog_

<a name="Exercise3" />
###Exercise 3: Saving an Image in the Image Library###

In this exercise you are going to use the capture feature of Windows Azure IaaS to create a new image based off of an existing virtual machine (the previously created one).

>**Note:** Before proceeding, ensure the **imgtest1** Virtual Machine is off. Wait until the sysprep finishes and turns off the Virtual Machine

<a name="Ex3Task1" />
#### Task 1: Saving an Image in the Image Library ####

1. Open the Windows Azure Portal and click **Virtual Machines**.

1. Select the Virtual Machine you prepared in the previous exercise.

	![Opening Virtual Machine Dashboard](images/opening-virtual-machine-dashboard.png?raw=true "Opening Virtual Machine Dashboard")

	_Opening Virtual Machine Dashboard_

1. Click **Capture** at the bottom of the screen.

	![Capturing Image](images/capturing-image.png?raw=true "Capturing Image")

	_Capturing Image_

1. In the **Capture virtual machine** page, set the **Virtual Machine Name** to _remotepsvm1_ and select **I have sysprepped the virtual machine**, then store it as an image.

	![capturedlg](images/capturedlg.png?raw=true)

	_Image Details_


<a name="Exercise4"></a>
### Exercise 4: Provision New Virtual Machines based on the created Image ###

In this exercise you are going to create a new virtual machine using the image you created in exercise 3. 

> **Note:** Before proceeding wait until the image finished provisioning You can switch to Images tab under Virtual Machines to check the status of the image.

<a name="Ex4Task1" />
#### Task 1: Create a new Virtual Machine using a base-image ####

1. Log in to the Windows Azure Portal:  https://manage.windowsazure.com.

1. Click **New** | **Compute** | **Virtual Machine** | **From Gallery**.
	![createvmfromimage](images/createvmfromimage.png?raw=true)

	_Creating Virtual Machine from Image_

1. In the **Virtual Machine operating system selection** switch to **MY IMAGES** and select **you-image-name**

	![myimagetab](images/virtualmachineimageselection.png?raw=true)

	_Virtual Machine operating system selection_

1. Set the **Virtual Machine Name** to _remotepsvm1_ and complete the Administrator Password

	![myimagetab](images/virtualmachinename.png?raw=true)

	_Virtual Machine configurtation_

1. Set the **DNS Name** to _remotepsvm1_ and select the **Image** you created previously (e.g. _remotepsvm1_).

	![myimagetab](images/virtualmachinedns.png?raw=true)

	_Virtual Machine mode_

1. Leave the **Virtual Machine options** with the default value and complete the operation.

1. Once the Virtual Machine is provisioned, click **Connect** at the bottom of the screen. When prompted login with _Administrator_ credentials.

<a name="Ex4Task2" />
#### Task 2: Create an Endpoint to Allow Traffic to the Virtual Machine ####

1. Open the Windows Azure Portal from https://manage.windowsazure.com and click **Virtual Machines**.

2. Click on the **remotepsvm1** Virtual Machine to open its **Dashboard** and then click **Endpoints**.

	![Virtual Machine Endpoints](images/virtual-machine-endpoints.png?raw=true "Virtual Machine Endpoints")

	_Virtual Machine Endpoints_

4. Click **Add Endpoint** towards the bottom of the page, select **Add Endpoint** option and click **next**. 

5. In the **Specify Endpoint Details** page, use the following settings:

	![add-endpoint](images/add-endpoint.png?raw=true)


>**Note:** Before Proceeding Ensure the Endpoint has Completed Configuration

<a name="Ex4Task3" />
#### Task 3: Connect from a Remote Client using PowerShell ####

1. To connect remotely you will need some information regarding the Virtual Machine. To obtain this information, you will log in to the Windows Azure Portal and go to your Virtual Machine's **Dashboard**.

1. In the **Dashboard** page, locate and take note of the DNS.

	![Obtaining VM information](images/obtaining-vm-information.png?raw=true "Obtaining Virtual Machine information")

	_Obtaining Virtual Machine information_

1. If not already open, Go to Search tab in your machine and in the Search App box type Powershell.

1. Open **Windows Powershell** from the result page

1. Execute the following command to test the Virtual Machine connection. Update the placeholders with your Virtual Machines hostname. 
	
	<!-- mark:1 -->
	````PowerShell
	enter-pssession -computername [HOSTNAME].cloudapp.net -authentication Basic -Credential administrator -UseSSL -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck)

	````

1. To test that you were successfully connected to the Virtual Machine, you will run the following commands to see the remote file system. Notice that at the left of your prompt you can see the host name that you are connected to.

	````CMD
	cd\
	dir
	````

	![remote-prompt](images/remote-prompt.png?raw=true)

	_Remote File System_

---

<a name="Summary" />
##Summary##
By completing this Hands-on Lab you have learned how to:

 - Configure PowerShell for remoting via HTTPS 
 - Customize and capture a virtual machine to the image library
 - Run remote commands on a virtual machine in Windows Azure IaaS

<a name="Appendix" />
##Appendix##

>**Note:** To Configure the same scenario for a domain joined computer you will need to use -Authentication Negotiate instead of Basic.