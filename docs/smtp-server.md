# SMTP Server

Amazon Simple Email Service (SES) is a cloud-based email service that provides cost-effective, flexible and scalable way to keep in contact with their customers through email.

SMTP-enabled programming language, email server, or application can be used to connect to the Amazon SES SMTP interface. The following information and a set of SMTP credentials is needed to configure this email sending method.

- SMTP endpoint: **email-smtp.eu-west-3.amazonaws.com**
- Transport Layer Security (TLS): **Required**
- STARTTLS Port: **25**, **587** or **2587**
- TLS Wrapper Port: **465** or **2465**

Some credentials have been already configured for the project. It can be found in the project's repository: [credentials.csv](https://github.com/5gmeta/platform-config/blob/main/src/credentials/SMTP/credentials.csv). In AWS the user created is: [ses-smtp-user.20221028-131744](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/users/details/ses-smtp-user.20221028-131744)

More details about the SMTP implementation for 5GMETA.

- All the mails sent by the platform must use the following address: [no-reply@5gmeta-platform.eu](no-reply@5gmeta-platform.eu)
- There are no mailboxes for receiving e-mails
- Need to authenticate into the server. Every partner should fill the [credentials request](https://vicomtech.sharepoint.com/:x:/r/sites/5GMETA2/Documentos%20compartidos/WP3%20-%205G%20NETWORK%20FUNCTIONS%20AS%20A%20DATA%20PLATFORM/Working%20Documents/Repository%20Policies/5GMETA_devtools_members.xlsx?d=w8c950169c8ad40febc0d40fad6f3b373&csf=1&web=1&e=RDrUuV).
