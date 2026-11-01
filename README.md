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

## How to deploy this — order of operations

| # | Page (breadcrumb) | What to do | Why here |
|---|---|---|---|
| 1 | **All → System Definition → Script Includes → New** | name **`IncidentUtils`**, *Client callable* **true**, *Accessible from* **All application scopes**, Active | the client script cannot call it before it exists |
| 2 | (same record) paste `getOpenIncidentCount()` | validate `sys_id` with `/^[0-9a-f]{32}$/i`, `GlideAggregate` COUNT on `incident` where `assigned_to` + `active = true`, return `'-1'` on bad input | cheapest place to reject bad input |
| 3 | **All → System UI → Client Scripts → New** | table **Incident**, type **onChange**, field **Assigned to**, paste `onChange(control, oldValue, newValue, isLoading, isTemplate)` | it fires when the field commits — i.e. exactly "assignee changed" |
| 4 | (same record) guards | `if (isLoading || isTemplate || !newValue) return;` | skips form load, template application and unassignment |
| 5 | (same record) stale-response guard | `if (g_form.getValue(fieldName) !== newValue) return;` inside the callback | two quick assignee changes out of order would show the wrong count |
| 6 | Incident form → test | assign a user who has > 10 active incidents | the only real test |

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
