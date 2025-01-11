---
title: 'Securing Wedding Website with Nginx'
date: '2025-01-11T13:55:47-05:00'
author: Guy Lewin
layout: post
tags:
  - nginx
  - authentication
  - web-security
  - wedding-tech
  - url-parameters
  - cookies
---

While planning my wedding, I found myself diving into an interesting technical challenge: how to share our Save the Date website with guests while keeping it private from the general public. This post details how I implemented a simple yet effective authentication system using Nginx, URL parameters, and cookies - without needing to modify the actual website content.

# The Challenge

My partner created a beautiful HTML-based Save the Date website that we wanted to share exclusively with our wedding guests. While the content wasn't particularly sensitive, I preferred to keep it private and away from search engines. The catch? I wanted to maintain the website as pure HTML without adding any authentication code to the frontend.

# The Solution

I developed a solution using Nginx that combines three key elements:

1. A password embedded in the URL as a query parameter
2. Browser cookie persistence for a smoother user experience
3. Server-side validation using Nginx configuration

When guests receive our Save the Date link (e.g., `https://ourwedding.com/save-the-date?pwd=secretkey`), Nginx validates the password, sets a persistent cookie, and redirects them to the clean URL. Future visits are authenticated via the cookie, eliminating the need for the query parameter.

# Implementation Details

For this example, let's use `aabbccddeeffgg` as our URL-friendly password. Here's the Nginx configuration that makes it all work:

```
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /\n";
}
location ^~ /save-the-date {
    # Disable client-side caching 
    expires -1;
    # Set password here
    set $password "aabbccddeeffgg";
    # Start with empty auth state variable
    set $auth_state "";
    # Auth state will be set to "q" if query string is correct, empty string otherwise
    if ($arg_pwd = $password) {
        set $auth_state "q";
    }
    # Auth state will be set to "qc" if cookie and query string correctly configured, "c" if only cookie, "q" if only query string and empty string if no correct authentication
    if ($cookie_pwd = $password) {
        set $auth_state "${auth_state}c";
    }
    # If there has been no correct authentication provided - return 403 Forbidden
    if ($auth_state = "") {
        return 403;
    }
    # If the query string was valid (with or without a cookie), set cookie and redirect to the URL without query string
    if ($auth_state ~ "qc?") {
        add_header Set-Cookie "pwd=${password};Domain=${host};Path=/save-the-date";
        return 301 /save-the-date;
    }
    root /var/www/ourwedding.com/public;
    try_files $uri /save-the-date/index.html;
}
```

Let's break down how this configuration works:

1. **Crawler Protection**: A standalone `/robots.txt` directive ensures search engines won't index the site, regardless of authentication status.

2. **Authentication Logic**: The main directive uses Nginx variables to track authentication state:
   - `$auth_state` can be empty, "q" (valid query string), "c" (valid cookie), or "qc" (both valid)
   - If no valid authentication exists (`$auth_state = ""`), returns 403 Forbidden
   - When a valid query string is present, sets the cookie and redirects to remove the parameter

3. **File Serving**: Once authenticated, Nginx serves files from the specified directory, defaulting to `index.html`.

# Future Plans

This configuration is part of a larger wedding RSVP system I'm developing. Once I've polished the complete solution, I'll be open-sourcing it for other couples who want to add a technical touch to their wedding planning.

Want to implement this for your own event? Just update the password, paths, and domain names in the configuration to match your needs. Just remember to choose a URL-friendly password - I recommend using a hex-encoded / Base64 string for compatibility.
```