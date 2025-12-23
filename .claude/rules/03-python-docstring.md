# Python Docstring Generation Rules

**When to use this file:** Reference this when writing or generating docstrings for Python functions, methods, and classes. Follows Google Python Style Guide.

**Related documentation:**
- For general code style, see [02-python-code.md](02-python-code.md)

## 1. Docstring Format
- **Style:** Adhere to the Google Python Style Guide for docstrings.
- **Quotes:** Always use triple double quotes (""") for all docstrings.
- **Structure:** The docstring should contain a one-line summary, followed by a blank line, and then an optional longer description that explains the overall purpose or behavior of the function in a concise and effective way.
- **Line Characters:** Each line is up to 88 characters long. It can wrap to multiple lines if needed.

## 2. Content Requirements
- **Summary:** The first line should be a concise summary (imperative mood: "Calculate" not "Calculates") of the function's or class's purpose.
- **Note Section:** Include an optional "Note:" section that lists important considerations, side effects, or special behavior that users should be aware of. For instance, "This function modifies the input object in place.", "This downloads all content into memory.", etc.
- **Arguments Section:** Include an "Args:" section that lists each argument.
- For each argument, specify its name, type, and a clear description.
- **Returns Section:** Include a "Returns:" section that describes the return value and its type.
- **Raises Section:** If the function can raise exceptions, include a "Raises:" section that lists each exception and the conditions under which it is raised.

## 3. Specific Instructions for Different Code Elements
### Functions and Methods
- For every function and method, generate a complete docstring.
- Infer parameter types from type hints whenever available.
- If a parameter has a default value, mention it in the description.
### Classes
- The class docstring should summarize the class's purpose and behavior.
- Include an "Attributes:" section to describe the class attributes.

## 4. Example of a Perfect Docstring
Here is an example of a well-formatted docstring that you should follow:
```python
def function(arg1: str, arg2: int = 10) -> list[str]:
    """Fetch and process data from a given API endpoint.

    Send a GET request to the specified API endpoint, parse the JSON response,
    and return a list of processed data.

    Note:
    - This request stores all data into memory, please ensure the server has
      sufficient memory.

    Args:
        arg1 (str): The URL of the API endpoint to fetch data from.
        arg2 (int, optional): The timeout for the request in seconds. 
            Defaults to 10.

    Returns:
        list[str]: A list of processed data items.

    Yields:
        str: A string representing a processed item, if this is a generator.

    Raises:
        requests.exceptions.RequestException: For connection errors.
        ValueError: If the response is not valid JSON.
    """
```
