# Setting Up a Custom Backup Schedule<a name="custom-backup-schedule"></a>

Amazon FSx for Windows File Server automatically takes a backup of your file system once a day during a daily backup window, and enforces a retention period that you specify on these automatic backups\. It also supports user\-initiated backups, so you can make backups at any point\.

Following, you can find the resources and configuration to deploy custom backup scheduling\. Custom backup scheduling performs user\-initiated backups on an Amazon FSx file system on a custom schedule that you define, such as once every 6 hours, once every week, and so on\. This script also configures deleting backups older than your specified retention period\.

The solution automatically deploys all the components needed, and takes in the following parameters:
+ The file system
+ A CRON schedule pattern for performing backups
+ The backup retention period \(in days\)
+ The backup name tags

For more information on CRON schedule patterns, see [Schedule Expressions for Rules](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) in the Amazon CloudWatch User Guide\.

## Architecture Overview<a name="fsx-custom-backup-overview"></a>

Deploying this solution builds the following resources in the AWS Cloud\.

![\[Image NOT FOUND\]](http://docs.aws.amazon.com/fsx/latest/WindowsGuide/images/fsx-custom-backup-architecture.png)

This solution does the following:

1. The AWS CloudFormation template deploys an CloudWatch Event, a Lambda function, an Amazon SNS queue, and an IAM role that gives the Lambda function permission to invoke the Amazon FSx API operations\.

1. The CloudWatch event runs on a schedule you define as a CRON pattern, during the initial deployment\. This event invokes the solution’s backup manager Lambda function that invokes the Amazon FSx `CreateBackup` API operation to initiate a backup\.

1. The backup manager retrieves a list of existing user\-initiated backups for the specified file system using `DescribeBackup` and then deletes backups older than the retention period, which you specify during the initial deployment\.

1. The backup manager sends a notification message to the Amazon SNS queue on a successful backup if you choose the option to be notified during the initial deployment\. A notification is always sent in the event of a failure\.

## AWS CloudFormation Template<a name="fsx-custom-backup-template"></a>

This solution uses AWS CloudFormation to automate the deployment of the Amazon FSx custom backup scheduling solution\. To use this solution, you'll need to download the [fsx\-scheduled\-backup\.template](https://s3.amazonaws.com/solution-references/fsx/backup/fsx-scheduled-backup.template) AWS CloudFormation template\.

## Automated Deployment<a name="fsx-custom-backup-deployment"></a>

The following procedure configures and deploys this custom backup scheduling solution\. It takes about five minutes to deploy\. Before you start, you must have the ID of an Amazon FSx file system running in an Amazon Virtual Private Cloud \(Amazon VPC\) in your AWS account\. For more information on creating these resources, see [Getting Started with Amazon FSx](getting-started.md)\.

**Note**  
Implementing this solution will incur billing for the associated AWS services\. For more information, see the pricing details pages for those services\.

**To launch the custom backup solution stack**

1. Download the [fsx\-scheduled\-backup\.template](https://s3.amazonaws.com/solution-references/fsx/backup/fsx-scheduled-backup.template) AWS CloudFormation template\. For more information on creating an AWS CloudFormation stack, see [Creating a Stack on the AWS CloudFormation Console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) in the *AWS CloudFormation User Guide*\.
**Note**  
By default, this template launches in the US East \(N\. Virginia\) AWS Region\. Amazon FSx is currently only available in specific AWS Regions\. You must launch this solution in an AWS Region where Amazon FSx is available\. For more information, see the Amazon FSx section of [AWS Regions and Endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html) in the *AWS General Reference\. *

1. For **Parameters**, review the parameters for the template and modify them for the needs of your file system\. This solution uses the following default values\.  
****    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/fsx/latest/WindowsGuide/custom-backup-schedule.html)

1. Choose **Next**\.

1. From **Options**, choose **Next**\.

1. From **Review**, review and confirm the settings\. You must check the box acknowledging that the template will create IAM resources\.

1. Choose **Create** to deploy the stack\.

You can view the status of the stack in the AWS CloudFormation console in the **Status** column\. You should see a status of **CREATE\_COMPLETE** in about five minutes\.

## Additional Options<a name="fsx-custom-backup-supplemental"></a>

You can use the Lambda function created by this solution to perform custom scheduled backups of more than one Amazon FSx file system\. The file system ID is passed to the Amazon FSx function in the input json for the CloudWatch event\. The default json passed to the Lambda function is as follows, where the values for `FilesystemId` and `SuccessNotification` are passed from the parameters specified when launching the AWS CloudFormation stack\.

```
{
	"start-backup": "true",
	"purge-backups": "true",
	"filesystem-id": "${FilesystemId}",
	"notify_on_success": "${SuccessNotification}"
}
```

To schedule backups for an additional Amazon FSx file system, simply create another CloudWatch event rule using the Schedule event source, the Lambda function created by this solution as the target\. Choose **Constant \(JSON text\)** under **Configure Input**\. For the JSON input, simply substitute the file system ID of the Amazon FSx file system to back up in place of $\{FilesystemId\} and either Yes or No in place of $\{SuccessNotification\} in the JSON above\.

Any additional CloudWatch Event rules you create manually will not be part of the Amazon FSx custom scheduled backup solution AWS CloudFormation stack, and so will not be removed if you delete the stack\.