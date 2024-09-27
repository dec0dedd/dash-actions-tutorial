# Hosting your own Dash app with Github Actions

You've made a Dash app with cool visualizations and now you want to share it with others? Github Pages is a great tool to do that! Here are the steps to
host your Dash app with Github Actions:

## 1. Set up a Github Repository

Create a public Github repository (like the one you see right now) and create a file e.g. `app.py` where the code behind the app will be.

## 2. Set up Makefile

To make working with the files easier let's create a Makefile. To do this create a file named `Makefile` in the root directory of your repository with the following code (replace all occurences of `dash-actions-tutorial` with the name of your repository):
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
4. (IMPORTANT!) We are going to host this app using Github Pages. If repository's name doesn't match author's name (in this case it doesn't, `dash-actions-tutorial` != `dec0dedd`) Github won't short the URL and the app will be hosted on 'https://dec0dedd.github.io/dash-actions-tutorial/'. This can be a problem, because Dash might start to look for files in the wrong directories. As an example, Dash might want to use file `_dash-component-suites/plotly/package_data/plotly.min.js`, but such file doesn't exist (because there is no such file as `https://dec0dedd.github.io/dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js`). Instead we want it to find file `dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js`, this way it will look for file in `https://dec0dedd.github.io/dash-actions-tutorial/dash-actions-tutorial/_dash-component-suites/plotly/package_data/plotly.min.js` (and will find it because it's there). To solve this problem we have to use `find` and `sed` commands to search through all the files to replace all occurences of `_dash-component-suites` with `dash-actions-tutorial/_dash-component-suites` (we will also do the same with directories `_dash-layout`, `_dash-dependencies`, `_reload-hash`, `_dash-update-component`, `assets`).
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