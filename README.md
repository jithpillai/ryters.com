# Ryters - Full-Stack Writing Platform (Cloudflare Architecture)

This document outlines the complete architecture for "Ryters," a full-stack writing platform built entirely on the Cloudflare stack.

# 1. Core Architecture

Here's how the services interact:

Frontend (UI): A Next.js app deployed on Cloudflare Pages. This provides the user interface (Dashboard, Content Pages, Create Form).

Backend (API): A Cloudflare Worker (running Hono for routing) acts as our secure backend API. It handles all business logic.

Authentication: Cloudflare Access is placed in front of the entire application (both the Next.js site and the API). This handles the Social Login (Google, Facebook) and issues a secure JWT.

Database: Cloudflare D1 (SQL) stores all structured data: user profiles, content (blogs, reviews, stories), comments, and likes.

File Storage: Cloudflare R2 stores all binary files, specifically the thumbnail images.

Caching: Cloudflare KV is used to cache expensive D1 queries, like the public dashboard feed or heavily-viewed articles.

# 2. Step-by-Step Setup Guide

Step 1: Set up Cloudflare Access (Social Logins)

This is the "login module" you requested, handled by Cloudflare itself.

Go to your Cloudflare Dashboard -> Zero Trust.

Go to Access -> Applications.

Click Add an application and choose Self-hosted.

Application Name: Ryters (Production)

Application domain:

Domain: ryters.yourdomain.com (This will be for your Next.js app)

Path: /* (Protect the whole site)

Add another domain:

Domain: api.ryters.yourdomain.com (This will be for your Worker API)

Path: /* (Protect the whole API)

Identity providers:

Enable Google and Facebook. You will need to follow the Cloudflare guides to provide a Google/Facebook Client ID and Secret.

Create your first policy:

Action: Allow

Session Duration: 24 hours (or as needed)

Include: Everyone (This means any user can try to log in).

Final Setup (in Application Settings):

Go to the Settings tab for your new application.

CORS: Add https://ryters.yourdomain.com to "Allow origins".

Cookies: Set SameSite: Lax and ensure HttpOnly is checked.

How it works: Now, any visit to ryters.yourdomain.com OR api.ryters.yourdomain.com will first be redirected to a Cloudflare login page. After logging in with Google/Facebook, Cloudflare will attach a secure, HttpOnly cookie named CF_App_Token to all future requests. Our API will validate this token to identify the user.

Step 2: Set up D1, R2, and KV

D1 (Database):

Run wrangler d1 create ryters-db

Get the database_id and add it to worker-api/wrangler.toml.

Run the schema: wrangler d1 execute ryters-db --file=./d1_schema.sql

R2 (Storage):

Run wrangler r2 bucket create ryters-thumbnails

Add the bucket_name to worker-api/wrangler.toml.

KV (Cache):

Run wrangler kv:namespace create RYTERS_CACHE

Add the id and preview_id to worker-api/wrangler.toml.

Step 3: Deploy the Backend API Worker

Navigate to the worker-api directory.

Run npm install.

Run wrangler deploy src/index.ts --name ryters-api.

Go to your Cloudflare Dashboard -> Workers & Pages -> ryters-api.

Add a Custom Domain: api.ryters.yourdomain.com. (This MUST match what you set in Cloudflare Access).

Step 4: Deploy the Next.js Frontend

Navigate to the nextjs-frontend directory.

Run npm install.

Connect your repository to Cloudflare Pages.

Build settings:

Framework: Next.js

Environment Variables: Add NEXT_PUBLIC_API_URL = https://api.ryters.yourdomain.com

Custom Domain:

Add the custom domain ryters.yourdomain.com. (This MUST match what you set in Cloudflare Access).

3. Core Features Implementation Notes

Superadmin: To make a user a superadmin, you must manually update their row in the users table in D1. Set their role from user to superadmin. The API middleware in worker-api/src/auth.ts (which you'll need to create) will check for this role on protected routes (like DELETE /admin/content/:id).

"Multiple Pages" for Novels: The content.body is a single TEXT field. To create "pages," instruct your users to add a simple separator like <!-- pagebreak --> in the text editor. Your Next.js frontend can then split the content: content.body.split('<!-- pagebreak -->') and render it with a "Next Page" button.

Edit-in-Place: On the content/[id].tsx page, fetch the content. Also, fetch the current user from useAuth(). If user.id === content.author_id, show an "Edit" button. Clicking this button should set a local state const [isEditing, setIsEditing] = useState(false), which will conditionally render the <CreateContentForm /> component (pre-filled with the content data) in place of the static content view.

Responsive Nav: The TopNav is standard. The BottomNav (in nextjs-frontend/components/BottomNav.tsx) can be a div with position: fixed; bottom: 0; that is only visible on mobile screens (e.g., md:hidden in Tailwind).

Dynamic Categories Nav: The API should have a GET /categories endpoint that queries the categories table. The Next.js layout can fetch this and render the navigation links.
