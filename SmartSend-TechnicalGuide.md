# Contents

- Installation
- Adding custom reports
- Adding custom attachments
- Extension example


# Overview

DXC Smart Business Form Email Manager for Microsoft Dynamics 365 Finance and Operations (or Smart Send) is a simple and smart solution for sending business forms and other reports to your customers and vendors.
To install Smart Send, first check to make sure you have the latest copy of the installation files.
The solution is delivered as a deployable package only.
Refer to the following link on how to Manage third-party models and runtime packages by using source control.

# Licensing
SmartSend License is managed using DXC License Manager.

## Adding Custom Reports
While out of the box functionality is for the business forms listed here, the solution can be extended on further business forms or reports. This technical guide describes how the DXC Smart Business Form Email Manager solution can be extended by adding custom report.

Refer to the sample model file that demonstrates how to extend SmartSend to support Request for Quote report (note the Request for Quote is now supported by DXC Smart Business Form Email Manager).

### Prerequisite
Minimum solution required is DXC Smart Business Form Email Manager Version 10.0.0.10018.

### To extend a new Development - New report format
To extend a new report format, there are 4 elements that need to be developed.


|    **Object**  |    **Changes**   |
|-|-|
| **Base enum** | Extend ECL_AutoPrintReport enum by adding an enum value. |
| **Action menu item** | Create a new action menu item. Make sure the enum parameters are set to your newly created enum value. |
| **Table extension** | Create a table extension based on your primary table Add the following 3 fields.  <br> * ESS_EmailSent <br> * ESS_SavedToBlobStorage <br> * ESS_BlobStorage_URL|
| **Class** | Create a new class that extends ESSReport class. Fill in the following methods. <br> * **getPrintMgmtDocumentType** – if the report that you wish to implement is part of print management. Then ensure this method is updated. <br> * **getEmailAddress** – Implement how to resolve the email address. <br> * **getReference** – Implement to return references that will be stamped on the email queue table. <br> * **updateBlobStorageFlag** – Implement to update the fields on the primary table to indicate it has been saved to blob storage. <br> * **updateFlag** – Implement to update the fields on the primary table to indicate it has been sent via smart send. <br> * **buildTokenMap** – Implement to have custom tokens. This is an optional method as smart send supports dynamic tokens. Refer to the user guide. Avoid the use of special characters when creating custom tokens. <br> * **sendEmailFromForm** – implement to allow the menu item on the form to work. Sample code is provided. If you have a report that is not part of print management, you may need to split the report accordingly. |


## Adding Custom Attachments
Smart Send uses the primary table and the email template to look for additional attachments, where document handling file type is specified against the business form in the parameters, and where the restriction of the file is set to External. Smart Send also includes an extension hook to enable a customer to create an extension class to attach a document handling attached file from another source, for example Purchase/Sales order lines. This technical guide provides sample code to achieve this for Purchase order lines.

### Prerequisite
Minimum solution required is DXC Smart Business Form Email Manager Version 10.8.32.10030.

### Add Purchase Order Line Attachments to PO Confirmation
Create an extension class for the ESSReport* class. Below is sample code on how to add purchase order line attachments to the email.


[ExtensionOf(classStr(ESSReportPurchPurchaseOrder))] <br>

public final class ESSReportPurchPurchaseOrder_Extension <br>

{
    public void additionalAttachment(SysOutgoingEmailTable _sysOutgoingEmailTable, ECL_AutoPrintReportMgmt _autoPrintReportMgmt)  <br>
    
    {
        next additionalAttachment(_sysOutgoingEmailTable, _autoPrintReportMgmt);  <br>
        
        if (_sysOutgoingEmailTable.ECL_RefTableId == tablenum(VendPurchOrderJour))  <br>
        {
            VendPurchOrderJour vendPurchOrderJour = VendPurchOrderJour::findRecId(_sysOutgoingEmailTable.ECL_RefRecId); <br>
            
            if (vendPurchOrderJour)
            {
                PurchLine purchLine;
            
                while select purchLine
                    where purchLine.PurchId == vendPurchOrderJour.PurchId
                {
                    if (purchLine)
                    {
                        SysEmailTable::essCopyAttachmentToOutgoingEmail(_sysOutgoingEmailTable.EmailItemId, _autoPrintReportMgmt.DocuFileType, purchLine.TableId, purchLine.RecId);
                    }
                }
             }
         }
      }
 } 


 ### Copy attachment method
Below is sample code on how to attach to an email.

public static void essCopyAttachmentToOutgoingEmail(

        SysEmailItemId  _emailItemId, 
        DocuTypeId      _docuType, 
        TableId         _refTableId, 
        RefRecId        _refRecId)
    {
        DocuRef                 docuRef;        
        SysOutgoingEmailData    outgoingEmailData;
        Filename                attachmentFileExtension;

        select firstonly outgoingEmailData
            order by DataId desc
            where outgoingEmailData.EmailItemId == _emailItemId;
        SysEmailDataId dataId = outgoingEmailData.DataId;

        while select docuRef
        where docuRef.RefTableId  == _refTableId && docuRef.RefRecId    == _refRecId &&
              docuRef.TypeId       == _docuType &&
              docuRef.Restriction  == DocuRestriction::External
        {
            if (docuRef.isValueAttached())
            {
                dataId++;
                outgoingEmailData.EmailItemId   = _emailItemId;
                outgoingEmailData.DataId        = dataId;
                outgoingEmailData.EmailDataType = SysEmailDataType::Attachment;
                outgoingEmailData.Data          = docuRef.getFileContentAsContainer();
                outgoingEmailData.FileName      =  docuRef.docuValue().FileName;
                attachmentFileExtension = '.' + docuRef.docuValue().FileType;
                outgoingEmailData.FileExtension = attachmentFileExtension;
                outgoingEmailData.insert();
            }
        }
   }
 
## Extension example
- [Extension example](SmartSendExtensionSample.zip) 
