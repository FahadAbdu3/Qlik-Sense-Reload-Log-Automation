# Qlik-Sense-Reload-Log-Automation


## Overview
This script automates dynamic email alerts for Qlik Sense server tasks, enhancing the monitoring of daily reloads. 
It is designed to notify administrators immediately after the completion of reload batches. 
This is particularly useful when applications are scheduled to run at various times throughout the day, making manual checks inconvenient.

## Key Features
- **Dynamic Email Alerts:** Configurable emails detailing the status of each task are sent after each reload batch.
- **Enhanced Notification Content:** The email includes:
  - A dynamic subject line that indicates the last reload date and the number of failed reloads.
  - A detailed table in the email body, formatted with HTML and CSS, lists application names, task names from QMC, reload statuses, timestamps, and the associated reload batch.
- **Backup File Generation:** An additional downloadable file containing the reload status table is created for backup and record-keeping purposes.

## How It Works
The application uses Qlik Web Connectors and the SMTP Connector API to parse the server log text file on the main admin server hosting Qlik Sense. 
The setup is inspired by and based on a thread from the Qlik Community, which has been further enhanced to include more detailed notifications and backup functionalities. 
The script is set to trigger in QMC > Tasks at specified intervals after the completion of set application reloads, ensuring timely and efficient monitoring.

## Contact
For any questions or further information, please feel free to reach out via LinkedIn or email.

## Acknowledgments
This project was inspired by discussions and a foundational article in the [Qlik Community](https://community.qlik.com/t5/Member-Articles/Setting-Up-Qlik-Sense-to-Send-Email-Notifications-on-Reload-Task/ta-p/1753023). 
Substantial modifications have been implemented to enhance its functionality to meet specific monitoring needs.

We hope this tool aids in the effective monitoring of Qlik Sense tasks, automating what would otherwise be a manual and tedious process. Thank you for your interest in Qlik-Sense-Reload-Log-Automation.
