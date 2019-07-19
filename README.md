### Blog structure

This blog is based upon the dual branch separation described [here](https://www.drewsilcock.co.uk/custom-jekyll-plugins). This basically means that one branch (source) contains the main folder for the blog while the second branch (master) is a nested git repository only containing the _site folder

Every time you change your blog and want to publish the changes you have to run the following commands:
<code>
jekyll build
cd _site
git add .
git commit
git push origin master
</code>

To just view the blog locally you run

jekyll serve

And then browse to the URL that is displayed in output window