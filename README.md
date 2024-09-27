# Hosting your own Dash app with Github Actions

Created a Dash app with impressive visualizations and ready to share it with others? GitHub Pages is an excellent platform for hosting your app! Follow these steps to deploy your Dash app using GitHub Actions:

## 1. Set up a Github Repository

Set up a public GitHub repository (similar to this one) and add a file `app.py` to store the code for your Dash app. In order to deploy your Dash app from Github Actions you need to update your repository's setting: go to `settings -> Pages` and set `Source` to `GitHub Actions` (under `Build and deployment`).

**DISCLAIMER**: Since Github Pages can only store static files, you need to use clientside callbacks. In order to turn a callback into a clientside callback, you need to rewrite it with JavaScript (example clientside callback has been put in `app.py`). More info on clientside callbacks can be found [here](https://dash.plotly.com/clientside-callbacks).

## 2. Set up Makefile

To make working with the files easier, create a Makefile. Add a file named Makefile to the root directory of your repository and include the following code (be sure to replace all instances of dash-actions-tutorial with your repository's name):
```
run_app:
	python3 app.py & sleep 30

	wget -r http://127.0.0.1:8050/
	wget -r http://127.0.0.1:8050/_dash-layout 
	wget -r http://127.0.0.1:8050/_dash-dependencies

	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dcc/async-graph.js
	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dcc/async-highlight.js
	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dcc/async-markdown.js
	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dcc/async-datepicker.js

	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dash_table/async-table.js
	wget -r http://127.0.0.1:8050/_dash-component-suites/dash/dash_table/async-highlight.js

	wget -r http://127.0.0.1:8050/_dash-component-suites/plotly/package_data/plotly.min.js

	mv 127.0.0.1:8050 pages_files
	ls -a pages_files
	ls -a pages_files/assets

	find pages_files -exec sed -i.bak 's|_dash-component-suites|dash-actions-tutorial\\/_dash-component-suites|g' {} \;
	find pages_files -exec sed -i.bak 's|_dash-layout|dash-actions-tutorial/_dash-layout.json|g' {} \;
	find pages_files -exec sed -i.bak 's|_dash-dependencies|dash-actions-tutorial/_dash-dependencies.json|g' {} \;
	find pages_files -exec sed -i.bak 's|_reload-hash|dash-actions-tutorial/_reload-hash|g' {} \;
	find pages_files -exec sed -i.bak 's|_dash-update-component|dash-actions-tutorial/_dash-update-component|g' {} \;
	find pages_files -exec sed -i.bak 's|assets|dash-actions-tutorial/assets|g' {} \;

	mv pages_files/_dash-layout pages_files/_dash-layout.json
	mv pages_files/_dash-dependencies pages_files/_dash-dependencies.json
	mv assets/* pages_files/assets/

	ps -C python -o pid= | xargs kill -9

clean_dirs:
	ls
	rm -rf 127.0.0.1:8050/
	rm -rf pages_files/
	rm -rf joblib
```

Let's quickly go over what the file does:

1. We run our app with `python3 app.py` and give it 30 seconds to fully load and generate all files,

2. We need to extract all necessary files from our app to be able to run it on Github Pages. We can do this with `wget` command since Dash hosts our app
on `http://127.0.0.1:8050/`.

3. We create a new folder named `pages_files` where we will store all necessary files to run the app. Now we just need to move all downloaded files to `pages_files` directory.

4. (IMPORTANT!) We are going to host this app using GitHub Pages. If the repository name doesn’t match the author's GitHub username (as in this case, where dash-actions-tutorial ≠ dec0dedd), GitHub won’t shorten the URL, so the app will be hosted at 'https://dec0dedd.github.io/dash-actions-tutorial/'. This can lead to issues, as Dash might search for files in incorrect directories. For example, Dash could try to access _dash-component-suites/plotly/package_data/plotly.min.js, but this file won't exist at https://dec0dedd.github.io/dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js. Instead, we need Dash to look for the file under dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js, so it will search https://dec0dedd.github.io/dash-actions-tutorial/dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js and find it. To fix this, we’ll use the find and sed commands to update all file paths by replacing instances of _dash-component-suites with dash-actions-tutorial/_dash-component-suites, and do the same for _dash-layout, _dash-dependencies, _reload-hash, _dash-update-component, and assets directories.

5. Now we just need to move remaining (modified files) to `pages_files` folder to add them to the artifact we'll create later.

## 3. Create a Github Actions workflow

Create directories `.github` and `.github/workflows`. Create a file `example-workflow.yml` in `.github/workflows` with the following code:
```yml
name: Deploy app
run-name: Deploy HTML pages generated from Dash app
on: [push]
jobs:

  run_app:
    name: Run app
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@main

      - name: Setup Pages
        if: github.ref == 'refs/heads/main'
        uses: actions/configure-pages@main

      - name: Setup python
        uses: actions/setup-python@main
        with:
          python-version: '3.12'
          architecture: 'x64'

      - name: Install project and dependencies
        shell: bash
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r requirements.txt
          export

      - name: Run Makefile files
        run: |
          make clean_dirs
          make run_app

      - name: Upload Artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-pages-artifact@main
        with:
          path: "./pages_files"

  deploy-pages:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: run_app
    
    permissions:
      pages: write
      id-token: write

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@main
```

Here is a quick description of what every part of the file does:

1. First we have to get access to repository code (to run the app). We can do this with [checkout](https://github.com/actions/checkout) action.

2. Then we need to set up necessary tools with actions:
    - Github Pages using [configure-pages](https://github.com/actions/configure-pages) action,
    - and Python using [setup-python](https://github.com/actions/setup-python) action,

3. Now we need to install all dependencies of our app.

4. Next we run Makefile commands (which will be described later) to generate all HTML/CSS/Js files required for the app to work.

5. We create and upload pages artifact with [upload-pages-artifact](https://github.com/actions/upload-pages-artifact) action.

6. Using previously created artifact we upload all generated files to Github Pages with [deploy-pages](https://github.com/actions/deploy-pages).

## 4. Your Dash app is ready!

If you've followed the steps correctly, your Dash app should now be live at https://{username}.github.io/{repo-name}/. As proof that these instructions work, I've created a sample Dash app in this repository. You can view it here: https://dec0dedd.github.io/dash-actions-tutorial/. For more advanced projects hosted on GitHub Pages and more ideas for what you can do with Dash, check out my other project, [Alcompare](https://github.com/dec0dedd/alcompare), where I compare and visualize different time series models using financial data. You can explore the hosted pages [here](https://dec0dedd.github.io/alcompare/).

If you have any questions feel free to email me or make an issue/pull request. Thanks for reading!