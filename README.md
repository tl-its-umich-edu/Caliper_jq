

# DEALING WITH CALIPER Data

## SOURCE ISSUES

### Unwrapped jsonl

Data is NOT in wrapper:

```json
  {
    "@context": "", "id":"", "type": ""
  }
```

```bash
cat INPUT | jq -c '{key/value pairs you want in the output - i.e: event_id: .id}' | jq -s '.' > OUTPUT
```

### Data is escaped json inside an "event" object:

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

### Event or events inside a "data" array:

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
