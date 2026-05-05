## Ansible-generic-sql-deployments
This repo is automates generic .ispac package installation, script execution and sql job execution through ansible

# Assume the follwoing manual process
Given that you only go take a step forward after successful installation / script or job execution - given that there exists a backup & rollback procedure 
1. Install the .ispac files in a given target server
2. Run the SQL scripts one after one only after success
3. Run the SQL jobs one by one
