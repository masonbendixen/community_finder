- Vault is under C:\Users\mason\Documents\Obsidian\CommunityFinder
- Created Vault by saying new Vault and choosing the Obsidian folder
- Switched to light mode
- Turn on Community Plugins
- - Installed and enabled:
	- Dataview
	- Quick Add
	- Plugin Update Tracker
	- Metadata Menu
	- Tasks
- Backlinks
	- Turned on show backlinks at the bottom of notes
- File Recovery
	- Changed History Length to 5000
- Property View - enabled core plugin
- Added Templates folder
- Added Claude folder
- Added Class folder
- Added Sources folder
	- Added Images subfolder
- Added Projects folder
- Changed Settings / Files and links / Default location for new notes to Projects
- Changed Settings / Files and links / Default location for new attachments to Sources/Images
- Changed Settings / Files and links / Excluded files to add Class, Templates, and Sources
- In QuickAdd Settings
	- Set the Template Folder Path to Templates
- Installed the minimal theme
- In Metadata Menu settings:
	- Under FileClass settings
		- Set Class Files path to Class/
		- Make sure to click Save icon next to this
	- In Class folder, create Project note
	- Right click on Project note tab and select Manage fields / Add new field
		- Field Name: Category
			- Field Type: Select
			- Add a value: Claude, Documentation, Implementation, Planning, Research, Source
		- Field Name: Status
			- Field type: Select
			- Add a value: Idea, Active, Complete, Cancelled, Delayed
- Under Templates
	- Add a file for each Category
	- Right click on the tab for the file and choose Add fileClass to filename
	- Select Project
	- Right click on the tab and choose Add missing fields at section...
		- Choose Add this field at the end of the frontmatter (Right most button)
	- Click on Category and choose the appropriate Category
	- Click on Status and default to Active
	- In frontmatter add:
		- Authors: Mason Bendixen
		- Last Updated: {{date:M/D/YYYY}}
		- Version: 0.1
		- tags: 
	- Add the various skeleton outline for this template
- In QuickAdd Settings, add a choice for each Template type to add the thing in the right folder
	- Do Add Choice for each category
		- Select Configure
		- Set template path to the md file under Templates for this category
		- Choose folder when creating a new note: Projects for everything but Source and Claude
		- Open - Open the created file
- Create a top level note called Dashboard
	- For each category, add:
	```
```dataview
TABLE WITHOUT ID
file.link AS Planning
FROM "Projects"
WHERE Category = "Planning"
SORT file.mtime DESC
LIMIT 10
```	
	```
- Adding it under GIT source control
	- Open a shell at the directory of the vault ("C:\Users\mason\Documents\Obsidian\CommunityFinder")
	- git init
	- Create .gitignore
```
# OS junk
.DS_Store
Thumbs.db

# Obsidian device-specific or transient files
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/appearance.json
.obsidian/app.json
.obsidian/cache/
.obsidian/plugins/obsidian-git/*-log.json

# Obsidian trash (if enabled)
.obsidian/trash/
.trash/

# Optional: if you want plugin *code* but not their caches, keep plugins/ and remove cache lines above.
# Keep snippets, themes, and plugin configs versioned:
# (snippets/, themes/, plugins/*/data.json are useful to sync settings)
```
- Added .gitattributes
```
*.md text eol=lf
*.css text eol=lf
*.json text eol=lf

# If you store big media, consider Git LFS:
# *.png filter=lfs diff=lfs merge=lfs -text
# *.jpg filter=lfs diff=lfs merge=lfs -text
# *.mp4 filter=lfs diff=lfs merge=lfs -text
```
- Created repository on Github
	- Log in to github
	- Create new repository
	- Repository name: community_finder
- Adding it under GIT source control (continued)
	- git config --global core.longpaths true
	- git add .
	- git commit -m "Initialize Obsidian vault"
	- git remote add origin https://github.com/masonbendixen/community_finder.git
	- git push -u origin master
- GIT Obsidian Plug In
	- Settings → Community plugins → install Git.
	- Configure:
		- Auto commit-and-sync interval (minutes): 5
		- Auto commit-and-sync after stopping file edits: Enable
	    - Commit message on auto commit-and-sync: `vault: {date}`.
	    - Pull on startup: Enabled
- Adding Levi
	- Install GIT
	- Install Obsidian
	- git clone https://github.com/masonbendixen/machine_learning.git
	- Turn on community plugins
	- Did it work? Yes it did.