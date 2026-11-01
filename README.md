/*

 onChange call happens when the field changes again, so we ignore the outdated response and let the newer call handle the new assignee.
 if (g_form.getValue(fieldName) !== newValue) {

Some Crucial Tips!
- if Scoped use  global.AbstractAjaxProcessor
- if Global use AbstractAjaxProcessor
- remember the OnChange function signature
- function onChange(control, oldValue, newValue, isLoading, isTemplate)
Argument	Purpose
control	Field control argument
oldValue	Previous value
newValue	New value
isLoading	Whether the form is loading
isTemplate	Whether a template is being applied
Interview rule: Use newValue to process the changed value, and use the loading/template flags to avoid unwanted execution.

getXMLAnswer() is asynchronous, watch out! ga.getAnswer is syncronous.



*/

/* 
Other methods:
Scope adjustment: The code above uses `global.AbstractAjaxProcessor`, appropriate for a scoped application. If `IncidentUtils` is in the global scope, use `AbstractAjaxProcessor` instead. Ensure the client-side GlideAjax name matches the Script Include's actual API name.

## 3. Important implementation notes

- `isTemplate` guard: Included to skip execution during template application.
- `!newValue`: Handles empty values and other falsy values.
- `control.name`: Avoids hard-coding `assigned_to` in the stale-response check.
- `g_form.clearMessages()`: Prevents old form messages from accumulating, but it clears all form messages, not just this warning. If other messages must remain visible, use a more targeted message-management strategy.
- Invalid response: The server returns `'-1'` for invalid input, and the client ignores negative counts.
- Open versus active: The query currently counts all incidents with `active = true`. Change the query if your business defines “open” by specific states instead.
- Security: Client-callable does not mean unrestricted access should be allowed. Apply appropriate authorization if workload counts are sensitive.

One final distinction: this implementation warns the user when the count exceeds 10; it does not prevent the assignment. A hard enforcement requirement needs server-side validation as well.


*/

/* *  a  GlideAjax implementation that:
1. client script onchange on incident form will trigger when assigned_to changes
2. a server-side script include will check for the assignee with >10 incidents ( however this should be handled dynamically for best practices)
3.	If so, display an info message warning the user about the assignee's workload
4.	Show the actual count in the message

*/



# glideajax-scriptinclude-onchange-incident-warning

GlideAjax Script Include (`IncidentUtils`) + onChange Client Script on `assigned_to`: warns the user
when the selected assignee already has more than 10 active incidents.
Warn only - it does not block the assignment (hard enforcement needs server-side validation).


Here is the consolidated, revised implementation. It incorporates the feedback while preserving the correct GlideAjax pattern and efficient server-side aggregation.

## 1. Client Script — onChange

Configuration: Table = `Incident`, Field name = `Assigned to`, Type = `onChange`.

```
function onChange(control, oldValue, newValue, isLoading, isTemplate) {
    if (isLoading || isTemplate || !newValue) {
        return;
    }

    var threshold = 10;
    var fieldName = control.name;

    var ga = new GlideAjax('IncidentUtils');
    ga.addParam('sysparm_name', 'getOpenIncidentCount');
    ga.addParam('sysparm_assignee', newValue);

    ga.getXMLAnswer(function(answer) {

        // Ignore the response if the field has changed
        if (g_form.getValue(fieldName) !== newValue) {
            return;
        }

        var count = parseInt(answer, 10);

        // Ignore invalid responses
        if (isNaN(count) || count < 0) {
            return;
        }

        // Clear previous form messages before displaying a new warning
        g_form.clearMessages();

        if (count > threshold) {
            var assigneeName = g_form.getDisplayValue(fieldName);

            g_form.addInfoMessage(
                assigneeName + ' currently has ' + count +
                ' active incidents assigned (threshold: ' +
                threshold + '). Please consider the workload.'
            );
        }
    });
}
```

## 2. Script Include — `IncidentUtils`

Configuration: Active = true; Client callable = true; Accessible from the calling application. Use the appropriate API name if the Script Include is scoped.

```
var IncidentUtils = Class.create();

IncidentUtils.prototype = Object.extendsObject(
    global.AbstractAjaxProcessor, {

        getOpenIncidentCount: function() {
            var assignee = this.getParameter('sysparm_assignee');

            // Validate the supplied user sys_id
            if (!assignee ||
                !/^[0-9a-f]{32}$/i.test(assignee)) {
                return '-1';
            }

            var ga = new GlideAggregate('incident');
            ga.addQuery('assigned_to', assignee);
            ga.addQuery('active', true);
            ga.addAggregate('COUNT');
            ga.query();

            var count = 0;

            if (ga.next()) {
                count = parseInt(
                    ga.getAggregate('COUNT'), 10
                ) || 0;
            }

            return count.toString();
        },

        type: 'IncidentUtils'
    }
);
```

Scope adjustment: The code above uses `global.AbstractAjaxProcessor`, appropriate for a scoped application. If `IncidentUtils` is in the global scope, use `AbstractAjaxProcessor` instead. Ensure the client-side GlideAjax name matches the Script Include's actual API name.

## 3. Important implementation notes

- `isTemplate` guard: Included to skip execution during template application.
- `!newValue`: Handles empty values and other falsy values.
- `control.name`: Avoids hard-coding `assigned_to` in the stale-response check.
- `g_form.clearMessages()`: Prevents old form messages from accumulating, but it clears all form messages, not just this warning. If other messages must remain visible, use a more targeted message-management strategy.
- Invalid response: The server returns `'-1'` for invalid input, and the client ignores negative counts.
- Open versus active: The query currently counts all incidents with `active = true`. Change the query if your business defines “open” by specific states instead.
- Security: Client-callable does not mean unrestricted access should be allowed. Apply appropriate authorization if workload counts are sensitive.

One final distinction: this implementation warns the user when the count exceeds 10; it does not prevent the assignment. A hard enforcement requirement needs server-side validation as well.

---

## Before you start — prerequisites checklist

Tick these off before step 1. Everything here is a hard prerequisite unless marked optional — the common
failures are a missing item from this list, not a step done wrong.

- [ ] Permission to create a **Script Include** and a **Client Script** (developer role or delegated admin)
- [ ] You know your **scope**: global app → `AbstractAjaxProcessor`; scoped app → `global.AbstractAjaxProcessor`.
      Choosing wrong throws at *call* time, not at save time
- [ ] Users can **read `incident`** — `GlideAggregate` respects ACLs, so an assignee who cannot read incidents gets
      `0`, not an error
- [ ] A **test user with 11 or more active incidents** for the final check
- [ ] Agreement on the **threshold** (`10`). If the business wants to tune it, plan a `sys_properties` entry now
      instead of editing script later

---

## How to deploy this — order of operations

Each step is one page to stand on: what to have ready first, the fields to fill, and the one suggestion that saves
rework.

### Step 1 — the Script Include
> **Breadcrumb:** `All → System Definition → Script Includes → New`

- **Prerequisite:** nothing, but know which scope you are in: a global app uses `AbstractAjaxProcessor`, a scoped app
  uses `global.AbstractAjaxProcessor`.
- **Fields to fill:** `Name` = `IncidentUtils` · `Client callable` = **true** (otherwise the client cannot see it at
  all) · `Accessible from` = *All application scopes* · `Active` = true · `Script` = `getOpenIncidentCount()`.
- **Suggestion:** keep the record global and the function narrow. *Client callable* means **any logged-in user** can
  invoke it, so return a count and nothing else — never a record list.
- **Checkpoint:** the Script Include saves; *Client callable* shows true.

### Step 2 — the server-side body
> **Breadcrumb:** the `Script` field of the record from step 1

- **Prerequisite:** step 1.
- **Fields to fill (what the code must do):** reject a bad `sys_id` with `/^[0-9a-f]{32}$/i` and return `'-1'` ·
  `GlideAggregate` + `addAggregate('COUNT')` on `incident`, encoded query `assigned_to=<sys_id>^active=true` ·
  answer = the count as a string.
- **Suggestion:** validate on the server even though the client "cannot" send junk — the endpoint is callable
  directly, and `'-1'` gives the client something it can distinguish from `0`.
- **Checkpoint:** hardcoding a known user's `sys_id` and calling it returns their open count.

### Step 3 — the onChange client script
> **Breadcrumb:** `All → System UI → Client Scripts → New`

- **Prerequisite:** step 1 saved and *Client callable* verified.
- **Fields to fill:** `Name` = Assignee workload warning · `Table` = **Incident** · `UI Type` = *All* (or Desktop) ·
  `Type` = **onChange** · `Field name` = **Assigned to** · `Active` = true · `Script` =
  `onChange(control, oldValue, newValue, isLoading, isTemplate)`.
- **Suggestion:** do not hard-code `'assigned_to'` — read it from `control.name`, so the same script works on
  *Assignment group* or any user-reference field later.
- **Checkpoint:** the script saves against the Incident table with the right field.

### Step 4 — the early-exit guards
> **Breadcrumb:** the `Script` field from step 3

- **Prerequisite:** step 3.
- **Fields to fill:** first line = `if (isLoading || isTemplate || !newValue) return;`.
- **Suggestion:** these three guards remove the three false alarms people report as bugs: a warning on every form
  load (`isLoading`), on every template application (`isTemplate`), and on unassigning (`!newValue`).
- **Checkpoint:** opening the form produces no message.

### Step 5 — the stale-response guard
> **Breadcrumb:** inside the `getXMLAnswer` callback

- **Prerequisite:** step 4.
- **Fields to fill:** `if (g_form.getValue(control.name) !== newValue) return;` before touching `g_form`.
- **Suggestion:** `getXMLAnswer()` is **asynchronous** — two fast assignee changes can come back out of order and show
  the count for the wrong person. This one line makes that impossible; `getXMLWait()` would also "fix" it by freezing
  the form, which is worse.
- **Checkpoint:** changing the assignee twice quickly never shows the first user's count.

### Step 6 — prove it on a real record
> **Breadcrumb:** any Incident form

- **Prerequisite:** a test user with **11 or more** active incidents.
- **Fields to fill:** none.
- **Suggestion:** test all three outcomes, not just the happy one — 11 incidents (message), 0 incidents (no message),
  empty field (no message, no console error).
- **Checkpoint:** `<name> currently has 11 active incidents assigned (threshold: 10)…` appears.

## Parameters to consider

| Parameter | Where | Effect |
|---|---|---|
| `threshold = 10` | client script | the warning boundary. Keep it a constant, or move it to a `sys_properties` entry so the business can tune it without a code change |
| the field | client script (`control.name`) | works on any user-reference field, not just `assigned_to` — do not hard-code the name |
| `active = true` | Script Include | "active" is not "open" in every process; swap in the state filter your business uses |
| *Client callable* | Script Include | required — and it also means any logged-in user can call it, so keep the answer non-sensitive |
| scope | Script Include | scoped app → `global.AbstractAjaxProcessor`; global app → `AbstractAjaxProcessor`. A mismatch throws at call time |
| warn vs enforce | design | this **warns only**; hard enforcement belongs in a `before` business rule |

## Things to watch out for

- `getXMLAnswer()` is **asynchronous** — anything that needs `count` must live inside the callback. (`getAnswer()` is
  only valid after a blocking `getXMLWait()`, which freezes the form; avoid it.)
- `g_form.clearMessages()` clears **all** form messages, not only this warning.
- The server returns `'-1'` for invalid input; the client must ignore negative counts or it warns on `-1`.
- `parseInt` + `isNaN` before comparing, or an empty answer silently becomes `0`.
- Client-callable + `GlideAggregate` means a COUNT runs on every assignee change — trivial at 12 K incidents, worth
  caching if the table reaches millions of rows.

## Validate it (5 minutes, no code)

1. Give a test user **11 active** incidents.
2. Open any incident and set **Assigned to** to that user.
3. Expect: `<name> currently has 11 active incidents assigned (threshold: 10). Please consider the workload.`
4. Assign to a user with 0 incidents → **no** message (not an empty one).
5. Clear the field → no message and no console error (`isLoading` / `!newValue` guards).
6. Optional: change `threshold` to `5` and repeat — the message must fire at 6, proving the constant drives it.
