---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 7/11/2026
Version: 0.1
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I'm working on C:\Users\mason\Documents\Obsidian\Knotty Yoga\Claude\Splitting the server up into components.md. The code base that I am trying to split into components is at C:\Users\mason\source\repos\knottyyoga. I'm going to create a new project that will be a C++ web server using Crow very similar to the webserver at this path with an angular frontend also very similar to the angular webserver at this path. I will want to use the components that Splitting the server up into components is creating in this project. I probably should also componentize the angular frontend eventually but lets focus on the server for now.

Besides the components that we will reuse, I want a server pretty similar to the existing server with a testhelper, scheduled jobs process, gtest unit tests, a webserver, a database helper. I want a similar db schema directory, create_database.cpp, and rough layered architecture like the existing sever: database schema, table helpers / util code, business logic, and endpoints with Google tests to validate each component.

The basic idea of the website is that we will build out a gay community site. We will have a component that scans the Internet using AI at known locations and gathers events that are upcoming and places them into the database with images, description, location, date, etc. The site will then show upcoming events on the website and have a calendar. Eventually, we will expand functionality to let people post events with admin approval and let people search for events. We will also eventually do advertising.

For now, I want to focus on standing up a minimal server that we can start adding functionality to.

# Place plan here