# Proposal: rotate audit log

Author: stonezdj

Discussion: https://github.com/goharbor/harbor/issues/15653

## Abstract

The audit_log table grows rapidly sometimes, need to delete some history records for better query performance and the disk spaces.
When the number of records is greater than 100M, it might slow down related queries significantly. the table size of audit_log in some env might changed to 10-20GB, it has some impact on the database performance and need to rotate periodically.

## Background

The audit_log is used to record the image pull/push/delete operations in Harbor. administrators could retrieve the history of the operation log. In a typical large Harbor server, there might be a large amount pull request and small amount of push request, delete request. most of the request are pull requests to public repository. the audit_log of these request might consumes 90% of the request. the query to the audit_log cannot work properly because the records in audit_log is huge.

Because the audit log is stored in database table, it cost of amount DB IO time to write the audit_log, it is better to provide a configurable way to log these information either log it in the file system or log it in the database.

The previous implementation treat all event equally, but the annonymous pull request, it should be less important than the delete/push request from authenticated users. Harbor should provide a way to log these event with different level, for example log the annoymous pull request as with a debug level and log the delete/push request when the log level is info.

The current audit_log just log the push/pull/delete event related to the image, there are variaty of operation event need to log in the audit log, for example, add/remove the member of a project, change the project tag retention policy etc. there should be flexible and extensible implementation of the audit_log

The audit_log table because of it is large size, it requires the DBA to create a job to clean up it periodically and it also cause the historical data cannot be retrieved.

## Proposal

Provide an option to configure the audit log settings, for example, log level, and the output, the output could be audit.log or audit_log table or both. if the audit_log table is used, the audit_log table will be rotated periodically with configuration.

If the audit.log forward port is not configured. the audit log will be stored in the audit.log only

## Non-Goals

- Backup the audit_log table in database
- Extended the audit log format to log more information like the project member change, project olicy change, etc.

## Rationale

Log all event in the audit log table and periodically rotate the audit_log table to the audit.log file maybe an option, but it will be a lot of DB IO time. also the audit log file's event time is not accurate. it will record the event time in the audit_log table, but the audit.log file will be the real event time.

## Compatibility

The previous version's data in audit_log table could be rotated.

## Implementation

The implementation could be specified in two parts, the first part is the audit logger, the second part is the audit log rotate job.

### Audit logger

#### Init audit logger

```
// InitAuditLog redirect the audit log to the forward endpoint
func InitAuditLog(level syslog.Priority, logEndpoint string) error {
	al, err := syslog.Dial("tcp", logEndpoint,
		level, "audit")
	if err != nil {
		logger.Errorf("failed to create audit log, error %v", err)
		return err
	}
	auditLogger.setOutput(al)
	return nil
}

```

#### Log audit event in the audit log manager

```
// Create ...
func (m *manager) Create(ctx context.Context, audit *model.AuditLog) (int64, error) {
	if strings.EqualFold(audit.Operation, "pull") {
		log.AL.WithField("operator", audit.Username).
			WithField("time", audit.OpTime).
			Debugf("%s :%s", audit.Operation, audit.Resource)
	} else {
		log.AL.WithField("operator", audit.Username).
			WithField("time", audit.OpTime).
			Infof("%s :%s", audit.Operation, audit.Resource)
	}
	if config.AuditLogRetentionHour(ctx) == -1 {
		return 0, nil
	}
	return m.dao.Create(ctx, audit)
}
```

#### Purge the audit log table

In the harbor core main function, start a go function to purge the audit log table periodically.

```
	pkgAudit.Mgr.StartAuditLogPurger()
```
The StartAuditLogPurger function will purge the audit log table with the configurable interval.

```
// StartAuditLogPurger ...
func (m *manager) StartAuditLogPurger() {
	ctx := context.Background()
	interval := config.AuditLogPurgeInterval(ctx)
	go func() {
		for {
			// TODO: create redis lock make sure only one task running
			retentionHour := config.AuditLogRetentionHour(ctx)
			if retentionHour == -1 || retentionHour == 0 {
				continue
			}
			if err := m.Purge(ctx, retentionHour); err != nil {
				log.Errorf("failed to purge audit log, error %v", err)
			}
			time.Sleep(time.Duration(interval) * time.Second)
		}
	}()
}
```

The audit log purger could be implemented with jobservice, but it is might be complex and not easy to maintain.

### UI

There is an audit log configurable page.
Configure items:

1. Log level -- the audit log level, it could be INFO, DEBUG.

2. Log forward endpoint  -- the audit log forward endpoint, it could be syslog server. default is harbor-log:10514, it is the docker-compose syslog forward endpoint.

3. Audit log retention hour -- the audit log retention hour, default is 0, means no purge operation. -1 stands for no audit log in the audit_log table. this value could be changed by the administrator, and it could be loaded by the audit log purger.

4. Audit log purge interval (optional), default is 1 hour, it is a system level configuration. it is not configurable in the audit log config page.

## Open issues (if applicable)

The curren audit log implementation is syslog, it is not work as previous core.log or registry.log, which are log to stdout of the container. the audit log might need to configure in kubernetes. 
