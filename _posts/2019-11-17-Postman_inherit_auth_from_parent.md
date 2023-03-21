---
title: How to use single auth method for every request in a Postman collection
tags: Postman
redirect_from: "/Postman_inherit_auth_from_parent/"
---

If you are just like me tired of messing with authorization header for every request in a collection then you're welcome for a very helpful solution.

## Solution

All you have to do is:

- Click edit on a collection;
- Navigate to 'authorization' tab;
- Choose and set up an authorization type, that will be used for every request in this collection;

![postman-collection-authorization-tab](/assets/images/posts/postman-collection-authorization-tab.png){: .center-image }

Once it's done, you need to ensure that an authorization type of every request is set to `Inherit auth from parent`:

![postman-collection-authorization-tab](/assets/images/posts/postman-request-authorization-tab.png){: .center-image }

If everything has been done properly, you won't have that problem again.
