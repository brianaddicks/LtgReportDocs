# LTGReport

Currently, this script is designed solely to get Data from various sources and push that data to an Azure SQL database used for Tableau. It operates by looping through the directories in the Plugins directory. It calls each script in each Plugin directory in alphabetical order.  Each script is dot-sourced by the main script (LTGReport.ps1), so the order in which the plugin scripts operate is important.

The main script has been written to allow for different types of output, eventually it should replace Veeam/vCheck reports and even be able to upload data to ConnectWise, IT Glue, etc.