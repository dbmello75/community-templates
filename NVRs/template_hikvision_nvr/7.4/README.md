# Hikvision NVR HTTP — availability and clock alarms

This Zabbix 7.4 template provides four first-stage alarms:

1. NVR unreachable: reuse the linked **ICMP Ping** template's existing availability trigger. Import the official ICMP Ping template first. Set the host interface address to the same NVR as `{$HIKVISION_ISAPI_HOST}`. No Zabbix agent is installed on the NVR.
2. API authentication failed: deviceInfo or status returns HTTP 401/403 (credentials or permissions).
3. API data unavailable: three consecutive failed samples on either endpoint, no health samples for 5 minutes, or no valid clock samples for 5 minutes. Errors include timeout, HTTP failure, malformed XML and unexpected XML roots.
4. Clock out of sync: three consecutive valid samples exceed 180 seconds; recovers on a sample within tolerance.

The last three alarms require a fresh successful ping, preventing new cascaded alerts when the NVR is unreachable. API data depends on authentication, and clock depends on both API alarms. Existing events may remain until their trigger is reevaluated. Existing camera/disk discovery alerts are unchanged in this stage.

## Collection

`hikvision_nvr.get_info` and `hikvision_nvr.get_status` retain their keys and UUIDs but now use built-in **Script** items with Digest authentication, `HttpRequest` and `XML.toJson`. No external scripts are needed. Each runs every minute with a 10-second timeout. Their original XML-derived JSON roots are preserved for existing dependent items.

Login status items retain their keys: 0 = valid expected response, 1 = HTTP 401/403, 2 = other collection or format error. Repeated health values are stored every minute rather than discarded for an hour. `_http` in master-item test output exposes the HTTP code (0 for exceptions). Exception text and raw error bodies are not returned.

## Clock and daylight saving

`{$HIKVISION.TIME.MODE}` defaults to **us_eastern**, following the deployment assumption that the NVR already adjusts its displayed local clock for DST in Massachusetts. This mode deliberately ignores the timestamp's offset and compares local wall time against US Eastern time. It applies current US rules (since 2007): second Sunday in March, first Sunday in November, with transition instants calculated in UTC. It does not depend on the server/proxy operating-system timezone.

Thus `2026-10-03T10:02:11-05:00` matches a collection instant of `2026-10-03T14:02:11Z` in us_eastern mode. This is an explicit deployment policy, **not proof of a Hikvision firmware offset bug**. A wrong reported offset is not itself alarmed in this mode. For devices reporting a reliable ISO timestamp, set the host macro to **iso8601** to honor the offset, including DST. Other regions must use iso8601; do not use us_eastern outside US Eastern locations.

`{$HIKVISION.TIME.MAX.DIFF}` = `180` seconds. `{$HIKVISION.HEALTH.NODATA}` = `5m`; keep above the one-minute polling interval. Clock differences are absolute seconds measured against the midpoint of the HTTP request at the collecting server/proxy; synchronize that system with NTP. Invalid timestamps are discarded, and missing valid clock samples feed the API-data alarm.

## First import and validation

- This updates **Hikvision NVR by HTTP** in the repository. The separately uploaded **Hikvision camera by HTTP** is a different template; importing this does not replace it.
- Import the YAML with update/create enabled and **Delete missing disabled**. First test on a dedicated host; linking both templates can cause duplicate inventory field assignments and alerts.
- Configure `{$HIKVISION_ISAPI_HOST}`, `{$HIKVISION_ISAPI_PORT}`, `{$USER}`, `{$PASSWORD}` and confirm the ICMP interface target.
- Check the two Login status items and Clock difference in Latest data. Allow at least three polls for consecutive-sample alarms.
- In a test host, verify invalid credentials produce only authentication failure, blocked HTTP produces API data failure while ping works, and blocked ping produces the inherited availability alarm. Verify clock mode against an independently known timestamp before changing the NVR's clock.

Validation performed: YAML parsing; preserved existing item keys/UUIDs, discovery rules, dashboards and graphs; JavaScript tests using mocked Zabbix HTTP/XML objects for HTTP success, 401/403, 404/500/302, malformed XML, unexpected roots and timeout; clock tests for summer/winter, both 2026 DST transitions, strict ISO mode, drift and invalid dates. Native Zabbix import, Duktape execution and live NVR collection have **not** yet been tested.

Zabbix JavaScript reference: https://www.zabbix.com/documentation/7.4/en/manual/config/items/preprocessing/javascript/javascript_objects
