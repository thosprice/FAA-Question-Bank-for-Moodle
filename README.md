# FAA-Question-Bank-for-Moodle

The following is derived from https://claude.ai/chat/34d41199-65e1-4ced-9063-b6d0ad3c930e

Figures: Host public-facing version on Github. This is the definitive version.  Instructions to implement local hosting directed to whoever is running the BTHS server take the form "Please place this file Github_link on the local server".  Moodle XML can have links of the form ```html<img src="http://10.65.x.x/figures/Figure12.png" onerror="this.onerror=null;this.src='https://raw.githubusercontent.com/you/repo/main/figures/Figure12.png';"> ```to rely on the Github as a fallback until local becomes available.  

A link reference page for student reference in the case of broken links can be coded similarly to rely on either hosting location. The link reference page needs to be generated automatically from the manifest.  The link reference page itself can be hosted at multiple pages with coding similar to 
```html<p>
  <a id="fig1" href="http://10.65.x.x/figures/Figure1.png" target="_blank">Figure 1</a>
  <img src="http://10.65.x.x/figures/Figure1.png" style="display:none"
       onerror="document.getElementById('fig1').href='https://raw.githubusercontent.com/you/repo/main/figures/Figure1.png';">
</p>```
Both gClassroom and Moodle might strip this code

Moodle may not support ```html<img src="..." onerror="">```, 'If it gets stripped, the more durable version is a small site-wide script (via Moodle's admin "Additional HTML"/custom JS injection, if you have access to that setting) that scans the page for ```html<img>``` tags matching your local figure-URL pattern and attaches the fallback behavior at page-load time instead'

Action items: 1) explore fallback link URLs in our Moodle implementation.  
2) If this works, the local and Github URLs can both be automatically placed into the xml from the spreadsheet. This will only need to be updated when the local IP changes. 
3) create a script to retrieve figure manifest and create link reference page
4) Explore and set up standard procedure via 'Moodle's admin "Additional HTML"/custom JS injection' to modify URLs

Questions: Definitive question contents are stored in a spreadsheet. Github doesn't support everything we get from a gSheet.  That spreadsheet can be accessed by a local python script and gspread API to retrieve the contents and create an XML for upload to Moodle. The script performs validation 1) Every Figure reference in the spreadsheet actually exists in the repo, 2) No duplicate question IDs. 3) A basic duplicate/near-duplicate question-stem scan across the whole bank (even a simple text-similarity pass).

Action items: 1) make the category tags in the spreadsheet HTML/XML friendly and make sure there are no whitespace, capitalization, etc. differences that might propagate to the Moodle question bank.
2) The script will convert Figure names into links, given the spreadsheet + URL stubs pointing to Github or local repos.  Script can be customized based on how Moodle handles the onerror image fallback.  
3) Github repo contains a base config file with LocalBaseUrl and GithubBaseUrl, along with any other settings, URL stubs, etc.

XML generated for later upload to Moodle server can be archived / versioned on Github.  Commit label can record questions modified. Date is automatically stored. The list of commit labels provides a granular changelog.  Changelog tab on the spreadsheet can be more strategic and broad.
