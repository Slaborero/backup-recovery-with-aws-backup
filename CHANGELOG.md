# Changelog - AWS Backup Solution

Todos los cambios notables en este proyecto serán documentados en este archivo.

El formato está basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

---

## [Unreleased]

### Fixed - 2026-02-07 11:05 ART
- **Backup Schedules - Corrección de Sintaxis Cron**: Corregida sintaxis de cron expressions para compatibilidad con AWS Organizations
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Problema: El despliegue del 22 de enero falló con error "The provided policy document does not meet the requirements of the specified policy type"
  - Causa raíz: AWS Organizations NO soporta la sintaxis `cron(0 5 2-31 * MON-SAT *)` que combina rango de días del mes con días de semana
  - Solución aplicada:
    - DailyBackupSchedule: `cron(0 5 2-31 * MON-SAT *)` → `cron(0 5 ? * MON-SAT *)`
    - WeeklyBackupSchedule: `cron(0 5 2-31 * SUN *)` → `cron(0 5 ? * SUN *)`
  - Impacto: Los schedules ahora usan sintaxis válida para Organization Policies
  - Nota: Habrá solapamiento cuando el día 1 caiga en día de semana (se ejecutarán daily/weekly + monthly), pero esto es aceptable y no causa problemas
  - Recursos afectados: rOrgDailyByTagPolicy, rOrgWeeklyByTagPolicy, rOrgDefaultDailyBackUpPolicy, rOrgDefaultWeeklyBackUpPolicy
  - Commit: Pendiente

### Changed - 2026-01-26 18:45 ART
- **Documentación - Actualización de Schedules**: Actualizada documentación para reflejar schedules sin solapamiento
  - Archivos: 
    - `docs/03-IMPLEMENTACION/02-Configuracion-Politicas.md`
  - Cambios realizados:
    - Actualizado schedule de Policy 1 (Daily): `cron(0 5 2-31 * MON-SAT *)`
    - Actualizado schedule de Policy 2 (Weekly): `cron(0 5 2-31 * SUN *)`
    - Actualizado schedule de Policy 3 (Monthly 3m): `cron(0 5 1 * ? *)`
    - Actualizado schedule de Policy 4 (Monthly 180d): `cron(0 5 1 * ? *)`
    - Actualizado schedule de Policy 5 (Monthly 10y): `cron(0 5 1 * ? *)`
    - Actualizado schedule de Policy 6 (Default Daily): `cron(0 5 2-31 * MON-SAT *)`
    - Actualizado schedule de Policy 7 (Default Weekly): `cron(0 5 2-31 * SUN *)`
    - Actualizado parámetros SSM en sección de configuración
    - Agregada explicación de lógica de no-solapamiento en cada política
    - Actualizado flujo de backup para datos críticos
    - Actualizada sección de modificación de schedules con advertencia de mantener lógica
  - Razón: La documentación mostraba los schedules antiguos (Dom-Vie, Sábados, 4to Sábado) que fueron cambiados el 22 de enero. Ahora refleja correctamente la lógica de no-solapamiento implementada

### Changed - 2026-01-22 15:35 ART
- **Backup Schedules - Eliminación de Solapamiento**: Ajustados schedules para evitar ejecuciones simultáneas de diferentes tipos de backup
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Parámetros modificados:
    - `DailyBackupSchedule`: `cron(0 5 2-31 * MON-SAT *)` (Lun-Sáb, días 2-31)
    - `WeeklyBackupSchedule`: `cron(0 5 2-31 * SUN *)` (Domingos, días 2-31)
    - `MonthlyBackupSchedule`: `cron(0 5 1 * ? *)` (Día 1 de cada mes)
  - Lógica de no-solapamiento:
    - Día 1: Solo mensual (sin importar día de la semana)
    - Domingos (días 2-31): Solo semanal
    - Lun-Sáb (días 2-31): Solo diario
  - Razón: Evitar backups redundantes y optimizar uso de recursos. Si el día 1 cae Domingo, solo se ejecuta el mensual (más completo)
  - Aplicado a: OrgBackupPlanDaily, OrgBackupPlanWeekly, OrgBackupPlanMonthly, OrgBackupPlanMonthly3m, OrgBackupPlanMonthly10y, OrgBackupPlanDefaultDaily, OrgBackupPlanDefaultWeekly

### Changed - 2026-01-22 10:15 ART
- **Backup Plan Tags**: Cambiado tag de identificación de planes de backup
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Tag anterior: `project=aws-backup` (mismo valor para todos los planes)
  - Tag nuevo: `bkp-plan=<nombre-del-plan>` (valor único por plan)
  - Valores por plan:
    - OrgBackupPlanDaily: `bkp-plan=OrgBackupPlanDaily`
    - OrgBackupPlanWeekly: `bkp-plan=OrgBackupPlanWeekly`
    - OrgBackupPlanMonthly: `bkp-plan=OrgBackupPlanMonthly`
    - OrgBackupPlanMonthly3m: `bkp-plan=OrgBackupPlanMonthly3m`
    - OrgBackupPlanMonthly10y: `bkp-plan=OrgBackupPlanMonthly10y`
    - OrgBackupPlanDefaultDaily: `bkp-plan=OrgBackupPlanDefaultDaily`
    - OrgBackupPlanDefaultWeekly: `bkp-plan=OrgBackupPlanDefaultWeekly`
  - Aplicado a: `backup_plan_tags` y `recovery_point_tags` de cada política
  - Razón: Permitir identificación única de cada plan y sus recovery points para filtrado, reportes y troubleshooting
  - Parámetro `pTagKey` actualizado de `project` a `bkp-plan`
  - Parámetro `pTagValue` removido (ahora cada plan usa su propio valor hardcodeado)

### Added - 2026-01-19 16:30 ART
- **DeletionPolicy en Recursos Críticos del Vault Central**: Agregada protección contra eliminación accidental
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-central-backup-account.yaml`
  - Agregado `DeletionPolicy: Retain` y `UpdateReplacePolicy: Retain` a:
    - `rCentralAccountBackupKey` (KMS Key)
    - `rCentralAccountBackupKeyAlias` (KMS Alias)
    - `rCentralBackupVault` (Backup Vault)
  - Razón: Prevenir eliminación accidental de recursos críticos que contienen recovery points de todas las cuentas en ambas regiones (us-east-1 y us-east-2)
  - Impacto: Los recursos se retendrán incluso si el stack de CloudFormation es eliminado

### Fixed - 2026-01-19 14:40 ART
- **Organization Policy Copy Actions Region**: Corregida región del vault destino a us-east-2
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Problema: En el fix anterior se cambió incorrectamente la región de us-east-2 a us-east-1
  - Solución: Restaurar región us-east-2 como vault destino para DR cross-region
  - Razón: La copia debe ir a us-east-2 (región secundaria) para protección geográfica, no a us-east-1 (misma región primaria)
  - ARN destino: `arn:aws:backup:us-east-2:557270420251:backup-vault:AWSBackupSolutionCentralVault`

### Fixed - 2026-01-19 14:25 ART
- **Organization Policy Copy Actions Structure**: Corregido formato de copy_actions en política daily-by-tag
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Problema: Error "The provided policy document does not meet the requirements of the specified policy type" en rOrgDailyByTagPolicy
  - Causa raíz: ARN del vault central hardcodeado con formato incorrecto y falta de target_backup_vault_arn
  - Solución: 
    - Cambiar región de us-east-2 a us-east-1 en el ARN del vault central
    - Agregar campo `target_backup_vault_arn` con referencia al parámetro `pCentralBackupVaultArn`
    - Usar estructura correcta de copy_actions según documentación de AWS Backup Organization Policies
  - Razón: AWS Organizations requiere estructura específica para copy_actions en backup policies

### Changed - 2026-01-19 12:15 ART
- **OrgBackupPlanDaily Copy Actions**: Agregada copia cross-region a us-east-2
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Agregado `copy_actions` a la política `OrgBackupPlanDaily`
  - Destino: Vault central en us-east-2 (arn:aws:backup:us-east-2:557270420251:backup-vault:AWSBackupSolutionCentralVault)
  - Retención: 7 días (igual que retención local)
  - Razón: Protección geográfica para backups diarios, sin copiar a us-east-1 (solo DR en us-east-2)
  - Nota: Los backups quedan locales en cuentas miembro + copia en us-east-2

### Fixed - 2026-01-19 09:50 ART
- **Central Vault ARN con Multi-Región**: Corregido ARN del vault central para usar solo primera región
  - Archivo: `backup-recovery-with-aws-backup/_backup-recovery-with-aws-backup.yaml`
  - Problema: Error "Resource ARN is not valid" en member accounts
  - Causa raíz: El parámetro `AWSBackupCentralAccountRegion` contiene `us-east-1,us-east-2`, generando ARN inválido
  - Solución: Usar `!Select [0, !Split]` para extraer solo la primera región (us-east-1)
  - El ARN del vault central apunta a us-east-1, el vault en us-east-2 se usará para copias adicionales vía Organization Policies
  - Commit: bc41eb7
  - Estado: ✅ Despliegue completado exitosamente

### Fixed - 2026-01-19 09:15 ART
- **StackSet Regions Array Format**: Corregido formato de array de regiones en StackSets
  - Archivos modificados:
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-central-backup-account.yaml`
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-restore-test-plan.yaml`
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-restore-test-validation.yaml`
  - Problema: Error "expected type: String, found: JSONArray" en Regions
  - Causa raíz: Usar `- !Ref` creaba array anidado `[["us-east-1", "us-east-2"]]` en lugar de `["us-east-1", "us-east-2"]`
  - Solución: Cambiar de `Regions: - !Ref` a `Regions: !Ref` (sin guión)
  - El parámetro SSM StringList ya devuelve un array, no necesita ser envuelto en otro array
  - Commit: e70a740

### Changed - 2026-01-19 08:50 ART
- **SSM Parameter Migration para Multi-Región**: Migrado parámetro de región central a nuevo nombre con tipo compatible
  - Archivos modificados:
    - `backup-recovery-with-aws-backup/_backup-recovery-with-aws-backup.yaml`
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-central-backup-account.yaml`
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-restore-test-plan.yaml`
    - `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-restore-test-validation.yaml`
  - Cambios realizados:
    - Renombrado parámetro SSM de `/backup/central-account-region` a `/backup/central-account-region-v2`
    - Cambiado tipo de `String` a `StringList` en template principal
    - Actualizado tipo a `CommaDelimitedList` en los 3 StackSets que lo consumen
  - Razón: CloudFormation no permite cambiar el tipo de un parámetro SSM existente, requiere crear uno nuevo
  - Permite desplegar vault central en múltiples regiones: us-east-1, us-east-2
  - Commit: 8ccf73e
  - Estado: ✅ Despliegue completado exitosamente

### Fixed - 2026-01-16 14:30 ART
- **SSM Parameter Type Compatibility**: Corregido tipo de parámetro SSM para soportar múltiples regiones
  - Archivo: `backup-recovery-with-aws-backup/_backup-recovery-with-aws-backup.yaml`
  - Problema: Error "Types for SSM parameters [/backup/central-account-region] defined in CFN template and SSM are incompatible"
  - Causa raíz: El parámetro SSM se creaba como `Type: String` pero el StackSet lo consumía como `CommaDelimitedList`
  - Solución: Cambiado tipo de `AWSBackupCentralAccountRegionSSM` de `String` a `StringList`
  - Permite valores separados por comas (us-east-1,us-east-2)
  - Commit: 0a10ec2

### Changed - 2026-01-16 14:25 ART
- **Central Vault Multi-Region Support**: Modificado StackSet para soportar múltiples regiones
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacksets/stackset-aws-backup-central-backup-account.yaml`
  - Cambiado tipo de parámetro `StackSetCentralBackupAccountRegionParameterStore` de `String` a `CommaDelimitedList`
  - Permite que el parámetro SSM `/backup/central-account-region` acepte múltiples regiones separadas por comas
  - Parámetro SSM actualizado manualmente a: `us-east-1,us-east-2`
  - Resultado esperado: El vault central se desplegará en ambas regiones (us-east-1 y us-east-2)
  - Commit: f38d6df

### Fixed - 2026-01-16 10:40 ART
- **Report Plans Scope**: Configurado scope de reportes para incluir toda la organización
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-report-plan.yaml`
  - Problema: Los reportes solo mostraban backup jobs de la cuenta de backup (129654119448), no de toda la organización
  - Causa raíz: El campo `Accounts` no estaba configurado en `ReportSetting`, usando el default (cuenta actual)
  - Solución: Agregado `Accounts: ["ROOT"]` en `ReportSetting` de los 5 Report Plans
  - Resultado esperado: Los reportes ahora incluirán backup jobs de todas las cuentas de la organización (21 jobs vs 4 jobs actuales)
  - Verificación: DevOps tiene 8 jobs, Networking tiene 9 jobs, Backup tiene 4 jobs = 21 total
  - Commit: 02adbc4

### Fixed - 2026-01-15 17:05 ART
- **Pipeline Ejecuciones Duplicadas**: Deshabilitado PollForSourceChanges en Source stage
  - Archivo: `backup-recovery-with-aws-backup/_backup-recovery-with-aws-backup.yaml`
  - Problema: Pipeline tenía dos triggers (Webhook + PollForSourceChanges) causando ejecuciones duplicadas en paralelo
  - Solución: Agregado `PollForSourceChanges: false` en Configuration del Source action
  - Resultado: Solo el webhook de GitHub disparará el pipeline (una ejecución por push)

### Changed - 2026-01-15 15:50 ART
- **Políticas Default Safety Net**: Modificados resource_types en políticas default-daily y default-weekly
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Removido: `arn:aws:ec2:*:*:volume/*` (EBS volumes)
  - Agregado: `arn:aws:s3:::*` (S3 buckets)
  - Razón: Las AMIs de EC2 ya incluyen los volúmenes EBS, evitando backups duplicados. S3 buckets ahora incluidos en safety net con conditions para excluir recursos taggeados
  - Recursos ahora incluidos: EC2 instances, RDS databases/clusters, DynamoDB tables, EFS, FSx, Storage Gateway, S3 buckets
  - Nota: Probaremos si S3 soporta conditions en Organization Policies (no documentado oficialmente)

### Fixed - 2026-01-15 14:30 ART
- **Report Plans ScheduleExpression**: Removido campo `ScheduleExpression` inválido de Report Plans
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-report-plan.yaml`
  - Problema: CloudFormation falló con error "extraneous key [ScheduleExpression] is not permitted" en ReportDeliveryChannel
  - Causa raíz: AWS Backup Report Plans NO soporta scheduling a nivel de CloudFormation. El campo `ScheduleExpression` no es válido dentro de `ReportDeliveryChannel`
  - Solución: Removido parámetro `ReportSchedule` y todas las líneas `ScheduleExpression` de los 5 Report Plans
  - Los reportes se generan automáticamente por AWS Backup según su lógica interna (diariamente)
  - Nota: El path de S3 cambió de `BackupJob/Backup/crossaccount/...` a `BackupJob/Backup/{AccountID}/crossregion/...` (comportamiento de AWS, no configurable)

---

## [2026-01-14] - Redespliegue Completo

### Fixed - 2026-01-14
- **Report Plans OrganizationUnits**: Removido campo `OrganizationUnits` de los 5 Report Plans
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-report-plan.yaml`
  - Razón: AWS Backup Report Plans no acepta Root IDs (r-clmc) en el campo OrganizationUnits, solo acepta IDs de OUs (ou-xxxx)
  - Commit: de377af

### Changed - 2026-01-13
- **Organization Policies - Default Safety Net**: Modificadas políticas `default-daily` y `default-weekly` para usar `resource_types` con conditions
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Cambiado de `selections.tags` a `selections.resources` con ARNs completos por tipo de recurso
  - Agregado `conditions.string_not_like` para excluir recursos con tag `data-type-bkp`
  - Recursos incluidos: EC2 instances, EBS volumes, RDS databases/clusters, DynamoDB tables, EFS, FSx, Storage Gateway
  - S3 removido (no soporta conditions en Organization Policies)
  - Lógica: Las políticas default respaldan TODOS los recursos de los tipos especificados EXCEPTO los que tienen el tag `data-type-bkp` con cualquier valor
  - Commit: d68f0af

### Added - 2026-01-14
- **DeletionPolicy Retain**: Agregado `DeletionPolicy: Retain` y `UpdateReplacePolicy: Retain` a recursos críticos
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-member-account.yaml`
  - Recursos protegidos: `rMemberAccountBackupVault`, `rMemberAccountBackupKey`
  - Razón: Prevenir eliminación accidental de vaults y KMS keys que contienen backups
  - Commit: eb4795f

---

## [2025-12-16]

### Changed - 2025-12-16
- **Tags**: Migración de tag `Environment` a `env`
  - Archivos: Todos los templates CloudFormation
  - Razón: Estandarización de nomenclatura de tags según políticas corporativas

---

## [2025-12-11]

### Changed - 2025-12-11
- **Organization Policies Tag Values**: Actualización de valores de tag `data-type-bkp`
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/aws-backup-org-policy.yaml`
  - Valores actualizados según definiciones en `Json Definition/politica.csv`
  - Políticas afectadas: daily-by-tag, weekly-by-tag, monthly3-by-tag, monthly6-by-tag, monthly10a-by-tag

---

## [2025-11-28]

### Fixed - 2025-11-28
- **StackSet AccountFilterType**: Cambiado de `INTERSECTION` a `NONE`
  - Archivo: `backup-recovery-with-aws-backup/cloudformation/stacks/stackset-aws-backup-central-backup-account.yaml`
  - Razón: Bug conocido de AWS CloudFormation donde `INTERSECTION` causa cancelación automática de operaciones de StackSet

### Added - 2025-11-27
- **Central Vault Architecture**: Migración de vault central a cuenta dedicada
  - Cuenta Central Vault: 557270420251
  - Actualizado ARN del vault central en Organization Policy
  - Actualizado parámetros SSM con nueva cuenta

---

## Tipos de Cambios

- **Added**: Nuevas funcionalidades
- **Changed**: Cambios en funcionalidades existentes
- **Deprecated**: Funcionalidades que serán removidas
- **Removed**: Funcionalidades removidas
- **Fixed**: Corrección de bugs
- **Security**: Cambios relacionados con seguridad
