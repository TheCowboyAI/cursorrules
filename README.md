# cursorrules
Cowboy AI Cursor Rules

[Cursor AI](https://cursor.sh) is an AI-powered code editor. .cursorrules files define custom rules for Cursor AI to follow when generating code, allowing you to tailor its behavior to your specific needs and preferences.

In essense these have been described as "agents", "gpts" and other names that really aren't accurate.

These rules are Project Specific Instructions for the AI to use inside Cursor.

Cursor can also use MCPs (model context protocol) to use other LLM based tools.

## MCPs

MCPs are a way to use other LLM based tools inside Cursor.

This is a protocol offered by Anthropic and has become a defacto-standard for using other LLM based tools inside Projects.



## MCPs in Cursor

Cursor supports MCPs by using the `@` symbol followed by the name of the tool you want to use.

For example:

```
@brave/search
```

This will use the Brave Search API to search for information.


We can give specific instructions in our Software Development Life Cycle (SDLC) to the AI for how we wish it to use these tools.

Talk to ruler through composer using @ruler

