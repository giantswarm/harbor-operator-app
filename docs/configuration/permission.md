## Harbor user permissions

By default users without administrator priveleges can create projects and view logs.

On public projects all users will be able to see the list of repositories, images, image vulnerabilities, helm charts and helm chart versions, pull images, retag images (need push permission for destination image), download helm charts, download helm chart versions.

## User role in projects

Harbor manages images through projects. You provide access to these images to users by including the users in projects and assigning one of the following roles to them.

Limited Guest: A Limited Guest does not have full read privileges for a project. They can pull images but cannot push, and they cannot see logs or the other members of a project. For example, you can create limited guests for users from different organizations who share access to a project.

Guest: Guest has read-only privilege for a specified project. They can pull and retag images, but cannot push.

Developer: Developer has read and write privileges for a project.

Maintainer: Maintainer has elevated permissions beyond those of ‘Developer’ including the ability to scan images, view replications jobs, and delete images and helm charts.

ProjectAdmin: When creating a new project, you will be assigned the “ProjectAdmin” role to the project. Besides read-write privileges, the “ProjectAdmin” also has some management privileges, such as adding and removing members, starting a vulnerability scan.

## Project members permissions

The following table depicts the various user permission levels in a project.

Action	Limited Guest	Guest	Developer	Maintainer	Project Admin
See the project configurations	✓	✓	✓	✓	✓
Edit the project configurations					✓
See a list of project members		✓	✓	✓	✓
Create/edit/delete project members					✓
See a list of project logs		✓	✓	✓	✓
See a list of project replications				✓	✓
See a list of project replication jobs					✓
See a list of project labels				✓	✓
Create/edit/delete project labels				✓	✓
See a list of repositories	✓	✓	✓	✓	✓
Create repositories			✓	✓	✓
Edit/delete repositories				✓	✓
See a list of images	✓	✓	✓	✓	✓
Retag image		✓	✓	✓	✓
Pull image	✓	✓	✓	✓	✓
Push image			✓	✓	✓
Scan/delete image				✓	✓
Add scanners to Harbor					
Edit scanners in projects					✓
See a list of image vulnerabilities	✓	✓	✓	✓	✓
See image build history	✓	✓	✓	✓	✓
Add/Remove labels of image			✓	✓	✓
See a list of helm charts	✓	✓	✓	✓	✓
Download helm charts	✓	✓	✓	✓	✓
Upload helm charts			✓	✓	✓
Delete helm charts				✓	✓
See a list of helm chart versions	✓	✓	✓	✓	✓
Download helm chart versions	✓	✓	✓	✓	✓
Upload helm chart versions			✓	✓	✓
Delete helm chart versions				✓	✓
Add/Remove labels of helm chart version			✓	✓	✓
See a list of project robots				✓	✓
Create/edit/delete project robots					✓
See configured CVE whitelist	✓	✓	✓	✓	✓
Create/edit/remove CVE whitelist					✓
Enable/disable webhooks			✓	✓	✓
Create/delete tag retention rules			✓	✓	✓
Enable/disable tag retention rules			✓	✓	✓
Create/delete tag immutability rules				✓	✓
Enable/disable tag immutability rules				✓	✓
See project quotas	✓	✓	✓	✓	✓
Edit project quotas *					
* Only the Harbor system administrator can edit project quotas and add new scanners.

## Summary

Harbor system administrator (logging in as admin throguh the db), have the highest level of user privilege. They are able to set up projects, endpoints and replication rules through the UI. They can also manage other users and grant them the same Harbor system administrator privilges. 

Otherwise, managing users permissions generally is accomplished by assigning them to projects and then choosing their role within their project. Their role will dictate their permissions, as outlined by the table above. 