I currently have a book and reading tracking system hosted in Airtable, which also exports to a static reading history list on my site, which is powered by an old version of Jekyll. I want to port the hosting of that data to the Capacities app.

This article describes how the current system works: https://thegreata.pe/articles/2020/02/17/building-a-custom-reading-list-with-airtable/

And my current reading history lives here: https://thegreata.pe/reading/ — you should check your work against this. The jekyll server is running locally, and the local copy of that page is at http://localhost:4000/reading/.

I have exported my readings and books to two CSV files in this directory. Note that the image urls contained within will expire within a few hours of downloading, so please test them and let me know if you need me to fetch a new set of files.

The Capacities API docs live at https://developers.capacities.io/ The API key is in the .env file under CAPACITIES_API_KEY — do not put this in version control.

Do not push any updates to github or my site (i.e. pushes to master) while doing this work. Also do not commit to git, I will do this on my own.
