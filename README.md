

# DEALING WITH CALIPER Data

## GENERAL STRUCTURE OF THE EVENT

```json
{
   "@context": "http://purl.imsglobal.org/ctx/caliper/v1p1",
   "id": "urn:uuid:438ff0c6-1231-4a83-8084-23eb67feaf62",
   "type": "AssignableEvent",
   "actor": {},
   "action": "Submitted",
   "object": {},
   "eventTime": "2018-03-16T16:33:29.000Z",
   "edApp": {},
   "group": {},
   "membership": {},
   "session": {},
   "extensions": {}
 }
```

`type, action` are the main identifiers

`actor` contains details on the user (id, type), most of them unfortunately inside an `.extensions['com.instructure.canvas']` object

`object` will contain details on the `action` - again most of them unfortunately inside an `.extensions['com.instructure.canvas']` object

`edApp` will tell you what software emitted the event

`group` holds details on the context of the event (a course say for CLE, a problem set in LECCAP, or maybe a Course as well)

`membership` will explain relationship of the user to the context

`sessions` will detail session stuff

`extensions` will give you server url, user agent etc.

## DATA FILES

Can be gotten from the Student and the Instructor folders at:

https://umich.app.box.com/folder/47628402451

Use the ones that have `testN_raw.json` format.

## SOURCE ISSUES

I have seen 3 different shapes to Caliper `jsonl`, I guess they depend on how they were extracted.

### 1. Unwrapped jsonl

Data is NOT in wrapper:

```json
  {
    "@context": "", "id":"", "type": ""
  }
```

```bash
cat INPUT | jq -c '{key/value pairs you want in the output - i.e: event_id: .id}' | jq -s '.' > OUTPUT
```

### 2. Data is escaped json inside an "event" object:

```json
  {
    "id": "f8d311fa-3a2f-4a9a-a818-40f2bb428b90",
    "event_type": "NavigationEvent",
    "event_action": "NavigatedTo",
    "event_time": "2018-01-30 12:48:33 UTC",
    "event": {}
  }
```
The command below parses the event object then pipes the result to another command

```bash
cat INPUT | jq -c '.event | fromjson' | jq '.| {key/value pairs you want in the output - i.e. event_id: id, etc. }' | jq -s '.' > OUTPUT
```

### 3. Event or events inside a "data" array:

```json
{
"sensor": "http://umich-dev.instructure.com/",
"sendTime": "2018-03-08T13:40:34.691Z",
"dataVersion": "http://purl.imsglobal.org/ctx/caliper/v1p1",
"data": []
}
```
```bash
cat INPUT | jq -c '.data[]' | jq -s '.' > OUTPUT
```
Will produce an array of event objects.

## REPO CONTENTS

This README.md and a file with all identified event types. The json in that file contains the markers of the event type they specify.

For example:

```json
"syllabusUpdated": {
    "type": "Event",
    "action": "Modified",
    "object/type": "Document",
    "object/name": "Syllabus"
  }
```
says, in other words, that in order to find all syllabus update events, you need to match the events that have those values. The `/` in `"object/type"` and `"object/name"`is a shorthand way of saying:

```json
"object":{
  "type":"Document",
  "name":"Syllabus"
}
```

## SELECTING A SUBSET

Can select slices with a select as the first piped command. All further examples use the unwrapped format. With wrapped formats need to unwrap first:

If wrap is an object with escaped json:

`cat INPUT | jq -c '.event | fromjson'` and then pipe to the select command.

For example, this selects only events by user with login id of "exampleusername". Note the select syntax `select(.actor.extensions?` - the '?' means essentially `if exists` - not all keys will be available in all the events. This keeps `jq` from throwing errors.

```bash
cat INPUT | jq -c '.event | fromjson' | jq '.|select(.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="exampleusername") |  {key value pairs you want in the output}' | jq -s '.' > OUTPUT
```

If wrap is an array:

`cat INPUT | jq -c '.data[]'`  and then pipe to the select command.

For example, this selects only events by user with login id of "exampleusername" like above.

```bash
cat INPUT | jq -c '.data[]' | jq '.|select(.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="exampleusername") |  {key value pairs you want in the output}' | jq -s '.' > OUTPUT
```

If event is not wrapped then the select can be done without unwrapping.

For example, this selects only events by user with login id of "exampleusername" like above.

```bash
cat INPUT  | jq '.|select(.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="exampleusername") |  {event_id: .id}' | jq -s '.' >  OUTPUT
```

In the examples that follow, we are dealing only with unwrapped raw events. Below I will be selecting subsets and outputting objects - if you want the raw event in the selection, omit the object construction pipe (the `{key value pairs you want}` part)

i.e. to select a specific type of event, like submitted assignments

```bash
cat INPUT |  jq '.|select(.action=="Submitted" and .type=="AssessmentEvent") |  {key value pairs you want}' | jq -s '.' > OUTPUT
```

i.e. to select events only in one course, the weird select statement is because the course id is contained in several places depending on the event type.

```bash
cat INPUT | jq '.|select((.membership? | . .id?=="urn:instructure:canvas:course:17700000000207865") or (.group? | . .id?=="urn:instructure:canvas:course:17700000000207865") or (.object? | . .id?=="urn:instructure:canvas:course:17700000000207865")) |  {key value pairs}' | jq -s '.' > OUTPUT
```

i.e. to select events related to an attachments or files:

```bash
cat INPUT | jq '.|select(.object.name?=="attachment") |  {key/value pairs}' | jq -s '.' > OUTPUT
```

i.e. select events indicating a file access:

```bash
cat INPUT  | jq -c '.data[]' | jq '.|select(.object.name?=="attachment" and .action=="NavigatedTo") |  {event_id: .id}' | jq -s '.' >  OUTPUT
```

## SORTING ON A KEY

Add `'sort_by(.KEY_NAME) |.'` as the last command to get a a list sorte on the specific key)

i.e. this will select all events by user xxx and sort them by time:

```bash
cat INPUT  | jq '.|select(.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="xxx") | jq -s 'sort_by(.eventTime) |.' > OUTPUT
```

## SOME EXAMPLES

You can run these against the testX_raw.json files in the Box folder. The select logic is derived from the identification of the event as outlined in all_events_match.json.

All the queries have the following pipes:

unwrapping the event  `jq -c '.data[]'`

selecting the data `'.|select((.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="studenta@umich.edu") and (.object.name?=="attachment" and .action=="NavigatedTo"))`

specifying the attributes of the resulting objects `{att_id:.object.id, eventTime: .eventTime, course_id: .group.id}`

### Files

All the events associated with files viewed by studenta@umich.edu (user test4_raw.json):

```bash
cat INPUT | jq -c '.data[]' | jq '.|select((.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="studenta@umich.edu") and (.object.name?=="attachment" and .action=="NavigatedTo"))' | jq -s '.' > OUTPUT
```

All the events associated with files viewed by studenta@umich.edu (user test4_raw.json) - but this time we are just outputting file id, event time and course id:

```bash
cat INPUT | jq -c '.data[]' | jq '.|select((.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="studenta@umich.edu") and (.object.name?=="attachment" and .action=="NavigatedTo")) | {att_id:.object.id, eventTime: .eventTime, course_id: .group.id}' | jq -s '.' > OUTPUT
```

### Announcement list views

```bash
cat INPUT |  jq -c '.data[]' | jq '.|select(.action=="AssignableEvent" and .type =="Submitted")' | jq -s '.' > OUTPUT
```

### Assignments Submitted

By anyone anywhere

```bash
cat INPUT |  jq -c '.data[]' | jq '.|select(.action=="Submitted" and .type=="AssignableEvent")' | jq -s '.'  > OUTPUT
```

By studenta@umich.edu anywhere

```bash
|  jq -c '.data[]' | jq '.|select((.action=="Submitted" and .type=="AssignableEvent") and (.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="studenta@umich.edu"))' | jq -s '.'
```
By studenta@umich.edu  in a specific course

```bash
cat INPUT |  jq -c '.data[]' | jq '.|select((.action=="Submitted" and .type=="AssignableEvent") and (.actor.extensions? | . ["com.instructure.canvas"] | . .user_login?=="studenta@umich.edu") and .group.id=="urn:instructure:canvas:course:85530000000000031")' | jq -s '.'
```

By all students, anywhere, on a given day:

```bash
|  jq -c '.data[]' | jq '.|select(.action=="Submitted" and .type=="AssignableEvent" and .eventTime>="2018-03-16")' | jq -s '.'
```
