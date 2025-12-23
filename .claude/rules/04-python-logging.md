# Python Logging Rules

**When to use this file:** Reference this for structured logging standards including event/payload format, past-tense verb conventions, and error logging with tracebacks.

**Related documentation:**
- For general code style, see [02-python-code.md](02-python-code.md)

## 1. Logging Format

* **Structured Format:** All logs must use structured dictionary format passed to the `msg` parameter.
* **Key Fields:**

  * `event`: A short, descriptive message summarizing what happened.

    * Must start with a **verb in past tense** followed by the **object**.

      * ✅ *Example:*  `"Started stream query"`, `"Fetched user profile"`, `"Sent Slack message"`.  
      * ❌ Avoid reversed order or passive forms such as `"Stream query started"` or `"User profile fetched"`.
  * `payload`: Contains contextual or supplementary information.
  * `traceback`: Included only in error logs to capture exception details.
* **Style Consistency:** Keep keys in lowercase with underscores only if necessary (e.g., `response_data`).


## 2. Logging Levels and Usage

### Info Logs

* Use `logging.info()` for successful or expected operations.
* Structure:

  ```python
  logging.info(
      msg={
          "event": "Posted Slack message result.",
          "payload": {
              "response_data": response_data,
          },
      }
  )
  ```
* The `"event"` field describes the completed action in past tense.
* The `"payload"` field should contain contextual data relevant to the event.

### Error Logs

* Use `logging.error()` for errors, exceptions, or unexpected failures.
* Must include traceback information for debugging.
* Import the traceback module at the top of the file:

  ```python
  import traceback
  ```
* Structure:

  ```python
  logging.error(
      msg={
          "event": str(e),
          "payload":{
            "traceback": traceback.format_exc(),
          }
      }
  )
  ```

## 3. General Rules

* **No Plain String Logs:** Do not log raw strings or concatenated messages. Always use structured dictionaries.
* **Verb Style:** The `"event"` message must always begin with a verb in **past tense** (e.g., *Fetched data.*, *Processed request.*, *Posted message.*).
* **Payload Data:** Keep payloads concise. Avoid including large or sensitive data (e.g., full request bodies or tokens).
* **Consistency:** Use the same structure across all modules and services to simplify downstream log parsing.
