# 07 Permissions per object type

| Type (`--types`) | Graph endpoint (beta) | Application permission | Checked in the lab |
|---|---|---|---|
| `settingsCatalog` | `deviceManagement/configurationPolicies`, `…/settings`, `…/assignments` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `deviceConfigurations` | `deviceManagement/deviceConfigurations?$expand=assignments` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `compliancePolicies` | `deviceManagement/deviceCompliancePolicies?$expand=assignments,scheduledActionsForRule(…)` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `scripts` | `deviceManagement/deviceManagementScripts/{id}`, `…/assignments` | `DeviceManagementScripts.Read.All` | yes (13 Nov 2026) |
| `shellScripts` | `deviceManagement/deviceShellScripts/{id}`, `…/assignments` | `DeviceManagementScripts.Read.All` | yes (13 Nov 2026) |
| `remediations` | `deviceManagement/deviceHealthScripts/{id}`, `…/assignments` | `DeviceManagementScripts.Read.All` | yes (13 Nov 2026) |
| `assignmentFilters` | `deviceManagement/assignmentFilters` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `administrativeTemplates` | `deviceManagement/groupPolicyConfigurations`, `…/definitionValues?$expand=definition`, `…/definitionValues/{id}/presentationValues?$expand=presentation` (Graph allows only one $expand level), `…/assignments` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `importedAdmx` | `deviceManagement/groupPolicyUploadedDefinitionFiles/{id}` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `endpointSecurityIntents` | `deviceManagement/intents`, `…/settings`, `…/assignments` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `complianceScripts` | `deviceManagement/deviceComplianceScripts/{id}` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `featureUpdates`, `qualityUpdates`, `driverUpdates` | `deviceManagement/windowsFeatureUpdateProfiles` (`…Quality…`, `…Driver…`), `…/assignments` | `DeviceManagementConfiguration.Read.All` | yes (13 Nov 2026) |
| `autopilotProfiles` | `deviceManagement/windowsAutopilotDeploymentProfiles`, `…/assignments` | `DeviceManagementServiceConfig.Read.All` | yes (13 Nov 2026) |
| `enrollmentConfigurations` | `deviceManagement/deviceEnrollmentConfigurations`, `…/assignments` | `DeviceManagementServiceConfig.Read.All` | yes (13 Nov 2026) |
| `termsAndConditions` | `deviceManagement/termsAndConditions`, `…/assignments` | `DeviceManagementServiceConfig.Read.All` | yes (13 Nov 2026) |
| `notificationTemplates` | `deviceManagement/notificationMessageTemplates?$expand=localizedNotificationMessages` | `DeviceManagementServiceConfig.Read.All` | yes (13 Nov 2026) |
| `deviceCategories` | `deviceManagement/deviceCategories` | `DeviceManagementManagedDevices.Read.All` | yes (13 Nov 2026) |
| `roles` | `deviceManagement/roleDefinitions`, `…/roleAssignments` | `DeviceManagementRBAC.Read.All` | yes (13 Nov 2026) |
| `scopeTags` | `deviceManagement/roleScopeTags`, `…/assignments` | `DeviceManagementRBAC.Read.All` | yes (13 Nov 2026) |
| `appProtectionIos`, `appProtectionAndroid`, `appProtectionWindows` | `deviceAppManagement/iosManagedAppProtections` (`android…`, `windows…`)`?$expand=apps,assignments` | `DeviceManagementApps.Read.All` | yes (13 Nov 2026) |
| `appConfigDevices` | `deviceAppManagement/mobileAppConfigurations?$expand=assignments` | `DeviceManagementApps.Read.All` | yes (13 Nov 2026) |
| `appConfigApps` | `deviceAppManagement/targetedManagedAppConfigurations?$expand=apps,assignments` | `DeviceManagementApps.Read.All` | yes (13 Nov 2026) |
| `namedLocations` | `identity/conditionalAccess/namedLocations` (v1.0) | `Policy.Read.All` | yes (15 Nov 2026) |
| `conditionalAccess` | `identity/conditionalAccess/policies` (v1.0) | `Policy.Read.All` | yes (15 Nov 2026) |
| `applications` | `deviceAppManagement/mobileApps?$filter=…` (own apps, Microsoft's when assigned), `…/{id}`, `…/assignments` | `DeviceManagementApps.Read.All` | yes (15 Nov 2026) |
| `customAttributes` | `deviceManagement/deviceCustomAttributeShellScripts/{id}`, `…/assignments` | `DeviceManagementScripts.Read.All` | yes (15 Nov 2026) |
| `complianceSettingsCatalog` | `deviceManagement/compliancePolicies?$expand=settings`, `…/scheduledActionsForRule`, `…/assignments` | `DeviceManagementConfiguration.Read.All` | yes (15 Nov 2026) |
| `reusableSettings` | `deviceManagement/reusablePolicySettings` | `DeviceManagementConfiguration.Read.All` | yes (15 Nov 2026) |
| `appleEnrollmentProfiles`, `appleAdeTokens` | `deviceManagement/depOnboardingSettings`, `…/{id}/enrollmentProfiles` | `DeviceManagementServiceConfig.Read.All` | yes (15 Nov 2026) |
| `deviceManagementSettings`, `applePushCertificate` | `deviceManagement?$select=settings,intuneBrand`, `deviceManagement/applePushNotificationCertificate` | `DeviceManagementServiceConfig.Read.All` | yes (15 Nov 2026) |
| `vppTokens`, `managedGooglePlay` | `deviceAppManagement/vppTokens`, `deviceManagement/androidManagedStoreAccountEnterpriseSettings` | `DeviceManagementApps.Read.All` | yes (15 Nov 2026) |
| `compliancePartners`, `managementPartners`, `remoteAssistancePartners` | `deviceManagement/complianceManagementPartners`, `…/deviceManagementPartners`, `…/remoteAssistancePartners` | `DeviceManagementServiceConfig.Read.All` | yes (15 Nov 2026) |
| `autopilotDevices` (off by default) | `deviceManagement/windowsAutopilotDeviceIdentities` | `DeviceManagementServiceConfig.Read.All` | against a simulated Graph |
| `activationLock` (off by default) | `deviceManagement/managedDevices/{id}?$select=…,activationLockBypassCode` | `DeviceManagementManagedDevices.PrivilegedOperations.All` | against a simulated Graph |
| `authenticationMethodsPolicy`, `authenticationFlowsPolicy`, `authorizationPolicy`, `externalIdentitiesPolicy`, `securityDefaults` | `policies/…` | `Policy.Read.All` | yes (15 Nov 2026) |
| `deviceRegistrationPolicy` | `policies/deviceRegistrationPolicy` | `Policy.Read.DeviceConfiguration` | yes (15 Nov 2026) |
| `groupSettings` | `groupSettings` (v1.0) | `Directory.Read.All` | yes (15 Nov 2026) |
| `domains` | `domains` (v1.0) | `Domain.Read.All` | yes (15 Nov 2026) |
| group names in assignments | `groups/{id}` (v1.0) | `Group.Read.All` | yes (13 Nov 2026) |
| who changed what (with `--commit`) | `deviceManagement/auditEvents?$filter=activityDateTime ge …` | `DeviceManagementApps.Read.All` | yes (13 Nov 2026) |
| who changed Conditional Access and named locations | `auditLogs/directoryAudits?$filter=activityDateTime ge … and category eq 'Policy'` (v1.0) | `AuditLog.Read.All` | yes (15 Nov 2026) |
| restore (`restore --apply`) | `PUT`/`PATCH`/`POST` on the type, `…/assign` | the `…ReadWrite.All` of the type (`DeviceManagementConfiguration`, `…Scripts`, `…ServiceConfig`, `…Apps`, `…RBAC`, `…ManagedDevices`), on the separate app `IntuneConfigBackup.Restore` ([restore](04-restore.md)) | device configurations (14 Nov 2026); the other types against a simulated Graph |
| `--include-oma-values` | `deviceManagement/deviceConfigurations/{id}/getOmaSettingPlainTextValue(…)` | **`DeviceManagementConfiguration.ReadWrite.All`**: Read.All is refused with "Application must have one of the following scopes: DeviceManagementConfiguration.ReadWrite.All" | yes (13 Nov 2026) |

The permissions of `IntuneConfigBackup.Reader` ([Setup](02-setup.md#intuneconfigbackupreader-reads-intune)) were enough for every type in the lab (run of 15 Nov 2026: 38 Settings Catalog policies, 18 device configurations, 72 apps, 20 Conditional Access policies, 7 named locations, 11 enrollment configurations, 10 filters, 9 scope tags, 6 Autopilot profiles, 5 remediations, the Entra policies and domains, the Apple push certificate and a VPP token, and the rest; types without objects in the lab tenant were read without an error). A refused type is reported as `NOT BACKED UP – no permission (needs …)` and its files from the last run stay.
