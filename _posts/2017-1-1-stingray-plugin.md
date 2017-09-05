---
layout: post
title: How to create a cool plugin using Stingray
category: stingray
tags: [autodesk, stingray]
---

# Requirements

-   Skills: C/C++ and JavaScript
-   Ruby 2.0 or later: <http://rubyinstaller.org>.
    -   Rubygems SSL fix (if needed): <http://guides.rubygems.org/ssl-certificate-update>
-   [Node 6.11.2 LTS or later](https://nodejs.org/en/).
-   Git client: <https://git-scm.com/>
-   Visual Studio 2015 with Update 3 & Patch KB3165756
-   Roll your sleeves and some willingness...

# Fork it!

Go to https://github.com/AutodeskGames/stingray-plugin and fork the repo on the `master` branch.

![image](https://cloud.githubusercontent.com/assets/4054655/23757119/ee53e8a2-04b3-11e7-9fa5-e1f19e223fc9.png)

Or clone it:

> git clone git@github.com/AutodeskGames/stingray-plugin.git giphy-plugin

In order to create our sample project later, we'll open the sample project provided by default with the stingray-plugin example right now, so while it compiles, I'll explain a few things.

Open the project manager and add an existing project. Select the plugin sample shown below.

![image](https://cloud.githubusercontent.com/assets/4054655/23757131/f5e97e1a-04b3-11e7-99a2-d1e9040b0de2.png)

![image](https://cloud.githubusercontent.com/assets/4054655/23757166/0ad13ad4-04b4-11e7-89c3-1f3fc10b579c.png)

While our project is loading, lets look at the plugin folder structure.

-   `editor`: Proposed folder to put editor native extension source code. (i.e. C/C++ plugin sources)
-   `engine`: Proposed folder to put engine native extension source code. (i.e. C/C++ plugin sources)
-   `plugin`: Other plugin sources (plugin descriptor, editor extension, compiled editor and engine extenions, etc.)
-   `tools`: Various build tools downloaded by `spm`.

# Build the plugin

Open a shell in the Stingray plugin repo and run the following command:

> ./make.rb

Notice that `make.rb` downloaded some extra stuff for you, such as some CMake scripts, a few extra ruby tools and the Stingray plugin SDK. Ain't that great?

Now our plugin folder structure looks like:

-   `build`: Various build outputs (i.e. CMake generated solutions)
-   `cmake`: CMake helper scripts downloaded by `spm`
-   `editor`: Proposed folder to put editor native extension source code. (i.e. C/C++ plugin sources)
-   `engine`: Proposed folder to put engine native extension source code. (i.e. C/C++ plugin sources)
-   `plugin`: Other plugin sources (plugin descriptor, editor extension, compiled editor and engine extenions, etc.)
-   `tools`: Various build tools downloaded by `spm`.
-   `stingray_sdk`: Stingray editor and engine C/C++ header based plugin SDK downloaded by `spm`.

Now you have a basic Stingray plugin compiled and ready to be loaded. Just so you know, it won't do much yet.

At the end of this tutorial, the plugin folder structure will look like:

![image](https://cloud.githubusercontent.com/assets/4054655/23757178/18ca88ac-04b4-11e7-955b-5bd8a5c1f8ad.png)

# Give it a name

Before we go wild and try to load the plugin, let's give it a name. How do we do that? Simple! Take your best text editor and editor the plugin descriptor. But first, let's rename the file `plugin/example.stingray_plugin` to `plugin/giphy.stingray_plugin`. Now open that file in your favorite editor.

## What is a plugin descriptor?

A plugin descriptor, describe your plugin intentions. It tells the user what it is and what is can do through plugin extensions.

Let's look at an example:

```
// About
//
name = "Giphy"
description = "Stingray Giphy Integration - Because I am a fun guy!"
version = "1.0.0"
changelog = {
    "1.0.0": "First version"
}
author = {
    name = "Jonathan Schmidt"
    email = "jonathan.schmidt@autodesk.com"
    company = "Autodesk Inc."
    url = "http://giphy.com"
}
keywords = ["autodesk", "stingray", "plugin", "example"]

// Define plugin thumbnail
//thumbnail = "sample_project/thumbnail.png"

// Extensions
//
extensions = {

    // Define the Giphy viewer view.
    views = [
        //{ type = "panel" name = "giphy-viewer" path = "giphy-viewer" title = "Giphy Viewer" width = 725 height = 800 }
    ]

    // Define a menu item that will open the Giphy viewer.
    menus = [
       // { path = "Window/Giphy" view = "giphy-viewer" order = 359 }
    ]

    // Add an asset type to see GIF assets in the asset browser.
    asset_types = [
       // { type = "gif" category = "Images" icon = "sample_project/thumbnail.png" }
    ]

    // Define a project template extension that will show up in the project manager.
    templates = [
        //{ type = "project-template" path = "sample_project/sample.stingray_project" }
    ]

    // Map plugin resources
    //resources = [ { path = "giphy_resources" } ]

    // Add an GIF importer.
    imports = [
        //{ types = ["gif"] label = "Giphy" do = [ { type = "js" module = "giphy-importer" function_name = "importGiphy" } ] }
    ]

    // Load the engine extension in the editor engine instance.
    runtime_libraries = [
        {
            name = "giphy_plugin"
            paths = {
                win32 = {
                    dev = "binaries/engine/win64/dev/engine_plugin_w64_dev.dll"
                    debug = "binaries/engine/win64/debug/engine_plugin_w64_debug.dll"
                    release = "binaries/engine/win64/release/engine_plugin_w64_release.dll"
                }
            }
        }
    ]
}

// Dependencies
//
platforms = ["win64"]
dependencies = {
    "stingray" = "~1.9.0"
}
```

You'll notice 3 main sections.

### About

In that section you give information about what is your plugin.

### Extensions

In that section you tell what your plugin can do and which system it extends.

### Dependencies

Here we indicate what are the dependencies of our plugin.

##### Continue...

So, copy paste the plugin descriptor into yours.

# Platform vs Plugins vs Extensions?

A platform provide built-in systems that does specific things, such as, loading plugins, show some nice widgets, etc.

A **plugin** adds new workflows to an application. A system should be able to work without any plugin.

An extension extends existing systems and other plugins. As an example, a platform can provide menu support, so as an extension, a plugin developer should be able to add new menu item using plugin menu extensions. To define any plugin extensions, you simply add extension declaration to the plugin descriptors `extensions` fields.

# Giphy?

Yeah! You probably figure it out already, the _cool_ plugin will be making, is a Giphy integration for Stingray.

Giphy is a platform to share GIFs, check it out here http://giphy.com

## Goals

For our plugin will want to do:

1. Search for some giphies within Stingray.
2. Download some Giphy files.
3. Extract all frames of a Giphy as PNG.
4. We want the extraction to be fast, why not do it as a native plugin extension?
5. I do not want to write an image library to parse GIFs and write PNGs, so we'll be using [stb](https://github.com/nothings/stb) as an external library.
6. Since I want to render those Giphy in game too, we'll be doing a Giphy engine plugin that compiles, renders and playback those awesome GIFs.
7. To finish, we'll be creating a project template to allow a user to quickly create new project from our Giphy integration.

# TS/JS setup

Since we'll be using JavaScript instead of TypeScript for this demo, we'll need to change a few things in our `tsconfig.json` configuration.

#### `editor/typescript/tsconfig.json`

```json
{
    "compilerOptions": {
        "target": "es6",
        "module": "amd",
        "moduleResolution": "node",
        "noImplicitAny": false,
        "noImplicitReturns": true,
        "noUnusedLocals": true,
        "removeComments": true,
        "preserveConstEnums": true,
        "alwaysStrict": true,
        "allowJs": true,
        "checkJs": false,

        "types": [
            "requirejs"
        ],
        "typeRoots": [
            "node_modules/@types"
        ],

        "sourceMap": true,
        "baseUrl": "./",
        "rootDir": "./",
        "outDir": "../../plugin"
    },

    "include": [
        "../../stingray_sdk/editor_foundation/types.d.ts",
        "../../stingray_sdk/editor_foundation/stingray.d.ts",
        "**/*.ts",
        "**/*.js"
    ],

    "exclude": [
        "node_modules"
    ]
}
```

#### `cmake/CMakeMacros.cmake`

Include JS files in typecript project generation

```cmake
# Defines TypeScript projects to be compiled
macro(add_typescript_project target source_dir tsconfig)
	set(files ${ARGN})

	set(TYPESCRIPT_EXTS "*.ts" ".js" "*.json" "*.html" "*.css" "*.stingray_plugin")
	find_source_files_of_type("${TYPESCRIPT_EXTS}" files ${source_dir})

	set(tsp "${source_dir}/${tsconfig}")
	message("-- Generating typescript project ${tsp}...")

	set (tsp_stamp "${CMAKE_CURRENT_BINARY_DIR}/${target}.ts.stamp")
	add_custom_command(
		OUTPUT ${tsp_stamp}
		COMMAND "node" ./node_modules/typescript/bin/tsc -p ${tsp} --noEmitOnError --listEmittedFiles
		COMMAND ${CMAKE_COMMAND} -E touch ${tsp_stamp}
		WORKING_DIRECTORY "${REPOSITORY_DIR}"
		DEPENDS ${files}
		COMMENT "Compiling typescript project ${tsp}")

	add_custom_target(${target} ALL DEPENDS ${tsp_stamp} SOURCES ${files})
endmacro()
```

# Giphy Viewer

Ok, we have our plugin descriptor ready for business. Now lets start by creating a simple Giphy viewer.

## Basic viewer

First let's start with a basic viewer. To do so, we will declare a view extension in the plugin descriptor. Open the plugin descriptor in a text editor and add the follow view extension in the `views` `extensions`.

Every extension type will be defined under the `extensions` field with the _plural_ extension type name. Every extension type blocks is an enumeration (i.e. an `array`) of extension declarations. 

So for `views`, you add in `extensions` the following.

```
extensions = {
    views = [
        { /* view extension declaration */ }
    ]
}
```

So lets define the Giphy viewer as a panel view like so.

```
// Define the Giphy viewer view.
views = [
    { type = "panel" name = "giphy-viewer" path = "giphy-viewer" title = "Giphy Viewer" width = 725 height = 800 }
]
```

For more information on view declaration see, <http://help.autodesk.com/view/Stingray/ENU/?guid=__sdk_help_extend_editor_plugin_extensions_views_and_dialogs_html>.

Then start with this simple JavaScript view module. At the same level as the plugin descriptor, create a new file named `giphy-viewer.js`.

#### `giphy-viewer.js`
```js
define(function (require) {
    'use strict';

    const stingray = require('stingray');
    const m = require('components/mithril-ext');
    const ListView = require('components/list-view');
    const Toolbar = require('components/toolbar');
    const Textbox = require('components/textbox');
    const hostService = require('services/host-service');

    /**
     * Giphy manager to show searched giphy.
     */
    class GiphyViewer {

        /**
         * Construct the Giphy viewer.
         */
        constructor () {
            // TODO
        }

        /**
         * Render the viewer with all the Giphy.
         */
        render () {
            // TODO
            return [];
        }

        /**
         * Refresh the entire view with new results.
         */
        refresh () {
            this.giphyListView.refresh();
        }

        /**
         * Push new results to the Giphy list view and refresh it.
         * @param {GiphyResult[]} result
         */
        showResult (result) {
            this.giphies = result;
            return this.refresh();
        }

        /**
         * Import all the frames of the selected Giphy.
         */
        importFrames () {
            // TODO
        }

        /**
         * Download the original Giphy file on disk.
         * @param giphy
         * @param folder
         * @returns {*}
         */
        saveGiphy (giphy, folder) {
            // TODO
        }

        /**
         * Call a C++ native function to extract all frames as png files.
         * @param filePath
         * @returns {*}
         */
        extractFrames (filePath) {
           // TODO
        }

        /**
         * Entry point to compose view.
         */
        static view (ctrl) {
            return ctrl.render();
        }

        /**
         * Mount a new playground
         * @param {HtmlElement} container
         * @return {{component, noAngular: boolean}}
         */
        static mount (container) {
            let instance = m.mount(container, {
                controller: this,
                view: this.view
            });
            return { instance, component: this, noAngular: true };
        }

        /**
         * Create a search box component to do Giphy searches.
         * @param {function} searchModel - Functor to get and set the search query.
         */
        static createSearchBox (searchModel) {
            return Textbox.component({
                model: searchModel,
                focusMe: true,
                clearable: true,
                liveUpdate: true,
                placeholder: 'Enter search query...'
            });
        }

        /**
         * Create basic list view columns.
         */
        static createColumn (type, property, width, name, dataType = undefined, format = undefined, onClick = undefined) {
            return {
                type,
                dataType,
                property,
                format,
                onClick,
                uniqueId: property,
                defaultWidth: width,
                header: { text: name, tooltip: `Sort by ${name}`},
            };
        }

        /**
         * Create the Giphy list view
         */
        static createListView (items, columns) {
            return ListView.config({
                id: 'giphy-list-view',
                items: items,
                columns: columns,
                layoutOptions: ListView.toLayoutOptions({ size: 7, filter: '' }),
                tooltipProperty: 'name',
                typedNavigationProperty: '*',
                thumbnailProperty: "thumbnail",
                filterProperty: '*',
                defaultSort: {uniqueId: 'name', property: 'name', reverse: false},
                showLines: true,
                showHeader: true,
                showListThumbnails: true,
                allowMultiSelection: false,
                allowMousewheelResize: true
            });
        }
    }

    document.title = 'Giphy Viewer';

    // Initialize the application
    return GiphyViewer.mount($('.main-container')[0]);
});
```

At this point we should have a view that works but doesn't do much.

Let's add something to open up that view, so we can see what is going on.

To do so, by adding a [menu extension](http://help.autodesk.com/view/Stingray/ENU/?guid=__sdk_help_extend_editor_plugin_extensions_menus_html):

```
// Define a menu item that will open the Giphy viewer.
menus = [
    { path = "Window/Giphy" view = "giphy-viewer" order = 359 }
]
```

This menu extension will add a menu item under the Window root menu item.

![image](https://cloud.githubusercontent.com/assets/4054655/23757209/28a423c8-04b4-11e7-8a25-7dc6f15a514b.png)

Once your view is open, open the debugger tools using **CTRL+F12**, so we can catch and debug any occurring errors while developing that view. The dev tools are really useful to see any debugging messages or thrown exceptions.

Let's construct some widgets. In the viewer constructor, add the following.

```js
// Construct view elements:
/**
 * Store all fetched Giphy.
 * @type {GiphyResult[]}
 */
this.giphies = [];

/**
 * Debounced function to search for new Giphy.
 * @type {Function}
 */
this.search = _.debounce(q => giphyClient.search(q).then(result => this.showResult(result)), 200);
/**
 * Create Giphy search query model.
 * This model getter/setter gets called each
 * time the model is updated are requested.
 */
this.searchQuery = '';
this.searchQueryModel = q => {
    if (!_.isNil(q)) {
        this.searchQuery = q;
        if (_.size(this.searchQuery) > 2) {
            this.search(this.searchQuery);
        }
    }
    return this.searchQuery;
};
/**
 * Toolbar items
 * @type {[*]}
 */
this.toolbarItems = [
    { component: GiphyViewer.createSearchBox(this.searchQueryModel) },
    { img: 'arrows-refresh.svg', title: 'Search...', action: () => this.search(this.searchQuery) },
    { img: 'save.svg', title: 'Import Giphy frames (as PNGs)...', action: () => this.importFrames() }
];
/**
 * Create the Giphy list view component.
 */
this.giphyListView = GiphyViewer.createListView(() => this.giphies, [
    GiphyViewer.createColumn(m.column.name, 'name', 300, 'Name', m.dataType.string)
]);
```

Render them using mithril by editing the render method of the viewer.

```js
render () {
    return m.layout.vertical({}, [
        Toolbar.component({items: this.toolbarItems}),
        m('div', {className: "panel-fill"}, [
            m('div', {className: "fullscreen stingray-border-dark"}, [
                ListView.component(this.giphyListView)
            ])
        ])
    ]);
}
```

At this point we should be able to see a search box with an empty list view. 

Let's fill up that list view.

## Giphy Web API client

To achieve that, we'll need an editor module to query the Giphy web service.

Here, let's take a look at this web API: https://github.com/Giphy/GiphyAPI#giphy-api-documentation

Our Giphy client will be like:

#### `giphy-client.js`
```js
define(function (require) {
    'use strict';

    const stingray = require('stingray');
    const httpClient = require('common/http-client');

    const GIPHY_PUBLIC_API_KEY = 'dc6zaTOxFJmzC';

    /**
     * Giphy Web API client.
     */
    class GiphyClient {
        constructor () {
            this.giphyAPI = httpClient('http://api.giphy.com/v1/gifs');
        }

        /**
         * Fetch trending Giphies.
         * @returns {Promise.<GiphyResult>}
         */
        fetchTrending () {
            return this.giphyAPI.get('trending', {api_key: GIPHY_PUBLIC_API_KEY})
                .then(result => this.parseResult(result));
        }

        /**
         * Show for new Giphy and restrict to PG-13.
         * @param {string} query - Query keywords
         * @returns {Promise.<GiphyResult>}
         */
        search (query) {
            return this.giphyAPI.get('search', {
                q: query.replace(/[\s]/g, '+'),
                limit: 20,
                rating: 'pg-13',
                api_key: GIPHY_PUBLIC_API_KEY
            }).then(result => this.parseResult(result));
        }

        /**
         * Parse Giphy query results and returned internal Giphy result.
         * @returns {Promise.<GiphyResult>}
         */
        parseResult (result) {
            if (_.size(result.data) <= 0)
                throw new Error('Result has no data');

            return result.data.map(giphy => {
                if (!_.isObject(giphy.images))
                    throw new Error('Result has no images');
                let images = giphy.images;
                return {
                    id: giphy.id,
                    name: (giphy.caption || giphy.slug || giphy.id).replace(/[-]/g, ' '),
                    thumbnail: images.fixed_height_small.url,
                    url: images.original.url,
                    width: images.original.width,
                    height: images.original.height,
                };
            });
        }

        /**
         * Download a Giphy's GIF image.
         */
        download (id, url, dir) {
            let outputGifFilePath = stingray.path.join(dir, `${id}.gif`);
            let downloadClient = httpClient(url);
            return downloadClient.downloadFile(outputGifFilePath).then(() => outputGifFilePath);
        }
    }

    return new GiphyClient();
});
```

Then require the module in the giphy viewer:

```js
const giphyClient = require('./giphy-client');
```

## Giphy Viewer

Finally, we have something to fetch Giphy search results and show them.

At construction, lets fetch trending Giphy.
```js
// Get trending results
giphyClient.fetchTrending().then(result => this.showResult(result));
```

Then, once the result are fetched, we should see them in the list view already.

We can also search for new Giphy using the search box.

At this point we should be seeing something like:

![image](https://cloud.githubusercontent.com/assets/4054655/23757233/34c25094-04b4-11e7-8b64-43b908e92993.png)

Also, at this point you might be tired to always re-transpile your JS sources into JS source, so let start a watcher to do it for us:

> ./node_modules/typescript/bin/tsc -p editor/typescript/tsconfig.json -w

Each time we'll be modifying a source file, the TS compiler will re-transpile everything for us.

## Download and extract GIF frames

Now that we have a way to list Giphy, it would be nice if we could take a Giphy,
 save it locally and extract all its frame as PNG files so we can import them
 as textures in Stingray. We want this process to be as fast as possible, that said,
 we'll create a native plugin extension using C++ to do so.

We'll be using stb to load GIF and save all frames as individual PNG files.

To do so, we'll need to update the editor CMake script located at `editor/native/CMakeLists.txt`.

Fetch stb using CMake ExternalProject macro
```cmake
# Fetch stb as an external library.
include(ExternalProject)
set(STB_INSTALL_LOCATION "${PROJECT_BINARY_DIR}/external/stb")
set(STB_INSTALL_LOCATION_LIB_DIR "${STB_INSTALL_LOCATION}/lib")
list(APPEND CMAKE_ARGS "-D${CACHE_VAR}${CACHE_VAR_TYPE}=${${CACHE_VAR}}")
ExternalProject_Add(external_stb
    GIT_REPOSITORY https://github.com/jschmidt42/stb-cmake.git
    CMAKE_ARGS "${CMAKE_ARGS};-DCMAKE_INSTALL_PREFIX=${STB_INSTALL_LOCATION}"
    UPDATE_COMMAND ""
)
```

Then we need to tell the editor plugin project to depend on stb and where to find the include files.

```cmake
# Setup stb component
add_library(stb STATIC IMPORTED)
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_DEBUG "${STB_INSTALL_LOCATION_LIB_DIR}/debug/stb${LIB_SUFFIX}")
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_DEV "${STB_INSTALL_LOCATION_LIB_DIR}/dev/stb${LIB_SUFFIX}")
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_RELEASE "${STB_INSTALL_LOCATION_LIB_DIR}/release/stb${LIB_SUFFIX}")
add_dependencies(stb external_stb)
add_dependencies(${PROJECT_NAME} stb)
include_directories(${STB_INSTALL_LOCATION}/include)
target_link_libraries(${PROJECT_NAME} stb)
```

Now, let's make sure the VS solution is up-to-date by running
 
 > ./make.rb --no-build
 
The solution should be generated under `build/editor/win64/editor_plugin.sln`
 
Open this up in Visual Studio.
 
At this point you should be seeing some basic API integration. We'll be replacing this with new function
to load GIF and extract frames and save them as PNG.
 
Let's expose an native API to JavaScript with:
 
 ```cpp
 /**
  * Setup plugin resources and define client JavaScript APIs.
  */
 void plugin_loaded(GetEditorApiFunction get_editor_api)
 {
 	auto api = static_cast<EditorApi*>(get_editor_api(EDITOR_API_ID));
 	config_data_api = static_cast<ConfigDataApi*>(get_editor_api(CONFIGDATA_API_ID));
 	logging_api = static_cast<EditorLoggingApi*>(get_editor_api(EDITOR_LOGGING_API_ID));
 	eval_api = static_cast<EditorEvalApi*>(get_editor_api(EDITOR_EVAL_API_ID));
 
 	api->register_native_function("nativeGiphy", "extractFrames", &extract_frames);
 }
 
 /**
  * Release plugin resources and exposed APIs.
  */
 void plugin_unloaded(GetEditorApiFunction get_editor_api)
 {
 	auto api = static_cast<EditorApi*>(get_editor_api(EDITOR_API_ID));
 	api->unregister_native_function("nativeGiphy", "extractFrames");
 }
 ```
 
So now let's implement the `extractFrames` function:
 
 ```cpp
 /**
  * Extract GIF frames and save them to disk as PNG files.
  */
 ConfigValue extract_frames(ConfigValueArgs args, int num)
 {
 	if (num != 1)
 		return nullptr;
 	auto file_path_cv = &args[0];
 	std::string file_path = config_data_api->to_string(file_path_cv);
 
 	int w, h, frames;
 	auto frames_data = gif_load_frames(file_path.c_str(), &w, &h, &frames);
 
 	// Create config data array to return all generated PNG file paths.
 	auto result_file_paths = config_data_api->make(nullptr);
 	config_data_api->set_array(result_file_paths, frames);
 
 	unsigned int frame_size = 4 * w * h;
 	for (int i = 0; i < frames; ++i) {
 		char png_filename[256];
 		auto frame_data = frames_data + i * (frame_size + 2);
 		auto delay = (stbi__uint16)*(frame_data + frame_size);
 
 		size_t dot_index = file_path.find_last_of(".");
 		auto raw_name = file_path.substr(0, dot_index);
 
 		sprintf(png_filename, "%s_%02d.png", raw_name.c_str(), i);
 		auto image_written = stbi_write_png(png_filename, w, h, 4, frame_data, 0);
 		if (!image_written)
 			continue;
 
 		char generation_log_info[1024];
 		sprintf(generation_log_info, "Generated `%s` with frame delay %d", png_filename, delay);
 		logging_api->info(generation_log_info);
 
 		auto file_path_item = config_data_api->array_item(result_file_paths, i);
 		config_data_api->set_string(file_path_item, png_filename);
 	}
 
 	return result_file_paths;
 }
 ```
 
We'll be needing `#include <string>`.
 
stb loads GIF just fine, but only it's first frame, so lets add a function to load the GIF as a series of images and delays.
 
 ```cpp
 /**
  * Load GIF animations frames from the specified file path..
  */
 STBIDEF unsigned char *gif_load_frames(char const *filename, int *x, int *y, int *frames)
 {
 	typedef struct gif_result_t {
 		int delay;
 		unsigned char *data;
 		struct gif_result_t *next;
 	} gif_result;
 
 	FILE *f;
 	stbi__context s;
 	unsigned char *result;
 
 	if (!((f = stbi__fopen(filename, "rb"))))
 		return stbi__errpuc("can't fopen", "Unable to open file");
 
 	stbi__start_file(&s, f);
 
 	if (stbi__gif_test(&s)) {
 		int c;
 		stbi__gif g;
 		gif_result head;
 		gif_result *prev = nullptr, *gr = &head;
 
 		memset(&g, 0, sizeof(g));
 		memset(&head, 0, sizeof(head));
 
 		*frames = 0;
 
 		while ((gr->data = stbi__gif_load_next(&s, &g, &c, 4))) {
 			if (gr->data == (unsigned char*)&s) {
 				gr->data = nullptr;
 				break;
 			}
 
 			if (prev) prev->next = gr;
 			gr->delay = g.delay;
 			prev = gr;
 			gr = (gif_result*)stbi__malloc(sizeof(gif_result));
 			memset(gr, 0, sizeof(gif_result));
 			++(*frames);
 		}
 
 		STBI_FREE(g.out);
 
 		if (gr != &head)
 			STBI_FREE(gr);
 
 		if (*frames > 0) {
 			*x = g.w;
 			*y = g.h;
 		}
 
 		result = head.data;
 
 		if (*frames > 1) {
 			unsigned int size = 4 * g.w * g.h;
 			unsigned char *p;
 
 			result = (unsigned char*)stbi__malloc(*frames * (size + 2));
 			gr = &head;
 			p = result;
 
 			while (gr) {
 				prev = gr;
 				memcpy(p, gr->data, size);
 				p += size;
 				*p++ = gr->delay & 0xFF;
 				*p++ = (gr->delay & 0xFF00) >> 8;
 				gr = gr->next;
 
 				STBI_FREE(prev->data);
 				if (prev != &head) STBI_FREE(prev);
 			}
 		}
 	} else {
 		stbi__result_info result_info;
 		result = (unsigned char*)stbi__load_main(&s, x, y, frames, 4, &result_info, 0);
 		*frames = !!result;
 	}
 
 	fclose(f);
 	return result;
 }
 ```
 
Finally let's add stb include statements:
 
```cpp 
#define STB_IMAGE_IMPLEMENTATION
#define STB_IMAGE_WRITE_IMPLEMENTATION
#define STBI_ONLY_PNG
#define STBI_ONLY_GIF
 
#include <stb_image.h>
#include <stb_image_write.h>
```
 
We should be able to compile the plugin dll successfully. Let's do that.

```
3>------ Build started: Project: editor_plugin, Configuration: Dev x64 ------
3>  editor_plugin.cpp
3>     Creating library D:/test-plugin/build/editor/win64/lib/Dev/editor_plugin_w64_dev.lib and object D:/test-plugin/build/editor/win64/lib/Dev/editor_plugin_w64_dev.exp
3>  editor_plugin.vcxproj -> D:\test-plugin\plugin\binaries\editor\win64\dev\editor_plugin_w64_dev.dll
```

Since we've expose a new native API: `window.nativeGiphy.extractFrames(filePath: string):string[]`
 
Let's use it in the Giphy viewer we started earlier:
 
```js
/**
 * Import all the frames of the selected Giphy.
 */
importFrames () {
    let selectedGiphy = _.first(this.giphyListView.getSelection());
    if (!selectedGiphy)
        return Promise.reject('No Giphy selection');
    // Ask user where to save frames.
    return hostService.getFolder('Select where to save frames...', stingray.env.userDownloadDir)
        .then(folder => this.saveGiphy(selectedGiphy, folder))
        .then(savedFilePath => this.extractFrames(savedFilePath))
        .then(extractedFrameFilePaths => hostService.showInExplorer(extractedFrameFilePaths[0]))
        .catch(err => console.error(err));
}
/**
 * Download the original Giphy file on disk.
 * @param giphy
 * @param folder
 * @returns {*}
 */
saveGiphy (giphy, folder) {
    if (!folder)
        return Promise.reject('Invalid folder');
    return giphyClient.download(giphy.id, giphy.url, folder);
}
/**
 * Call a C++ native function to extract all frames as png files.
 * @param filePath
 * @returns {*}
 */
extractFrames (filePath) {
    // Dynamically load the native plugin DLL
    const nativePluginDllPath = require.toUrl('@giphy/binaries/editor/win64/dev/editor_plugin_w64_dev.dll');
    if (!stingray.fs.exists(nativePluginDllPath))
        throw new Error('Giphy editor native plugin does not exists at `' + nativePluginDllPath + '`. Was it compiled?');
    let pluginId = stingray.loadNativeExtension(nativePluginDllPath);
    // Call our native function.
    /** @namespace window.nativeGiphy */
    let paths = window.nativeGiphy.extractFrames(filePath);
    // We do not need the plugin anymore, let's dispose of it.
    stingray.unloadNativeExtension(pluginId);
    return paths;
}
```
 
We can now use the save button to download and extract frames. 

The file explorer will be opened up with the saved gif and all the pngs. Try dragging png into the asset browser. See it works, now try with the gif, it doesn't. :( We'll get back to this...

# Giphy Sample Project

Ok all this is getting very exciting!

We can get as many Giphy as we want, but would't it be awesome if we could get those Giphy on a TV running in the engine?

To do so, let's start building a sample project to test that out. Once finished, we'll be able to provide this 
 sample project to our user to create as many instances of this project as they like to try different things.

## Setup sample project

Name your sample project. Edit the file `plugin/sample_project/sample.stingray_project` to have something like:

```
name = "Giphy"
source_directory = "./"
data_directory = "../../build/sample_project_data"
default_startup_level = "sample"
description = "Create a sample project that demonstrates the usages of the Stingray Giphy plugin."
libraries = {}
reverse_forward_axis = true
version = "1.9.0.0"
```

Everyone love thumbnails, let's get a thumbnail for this project:

<https://draoqiiudbn5q.cloudfront.net/hackathon/thumbnail.png>

![thumbnail.png](https://draoqiiudbn5q.cloudfront.net/hackathon/thumbnail.png)

Save that thumbnail in `plugin/sample_project`

`thumbnail = "thumbnail.png"`

Since we have a thumbnail, let's reuse it for the plugin descriptor as well by pointing to it. No need for duplication:

```
// Define plugin thumbnail
thumbnail = "sample_project/thumbnail.png"
```

# Giphy TV

So we want to create a TV unit to show these funny Giphy.

First, create a sub folder `giphy_resources` under `plugin/sample_project`.

Drag this [FBX](https://draoqiiudbn5q.cloudfront.net/hackathon/giphy_tv.fbx) in `giphy_resources`.

<https://draoqiiudbn5q.cloudfront.net/hackathon/giphy_tv.fbx>

Let's add some unit script data in order to be able to edit some field in the editor for this resource in the editor. Open the `.unit` file that was added when we dragged the FBX file in your text editor. Add the following:

We want the user be able to specify which `giphy_resource` to be shown on which `giphy_mesh_index` for a given `giphy_material_slot_name`.

```
data = {
	giphy_resource = ""
	giphy_mesh_index = 1
	giphy_material_slot_name = "color_map"
}
```

In order to present this information properly in the property editor let's add some script data UI controls:

```
editor_metadata = {
	data_ui = {
		categories = {
			giphy = {
				label = "Giphy"
				order = 80
			}
		}
		controls = {
			giphy_resource = {
				category = "giphy"
				extension = "gif"
				label = "Giphy"
				order = 1
				type = "resource"
			}
			giphy_mesh_index = {
				category = "giphy"
				label = "Mesh index"
				order = 2
				type = "number"
			}
			giphy_material_slot_name = {
				category = "giphy"
				default = "color_map"
				label = "Material Slot Name"
				order = 3
				type = "string"
			}
		}
	}
}
```

At this point, if you drag the TV in a level, you should see in the property editor the editable fields. Note that in the screenshot below the resource `giphy_resources/stingray` has been selected, we'll add this resource later:

![image](https://cloud.githubusercontent.com/assets/4054655/23757288/58ab3926-04b4-11e7-93bf-3826bae6c250.png)

Ok our TV looks good at this point.

## Screen material

First, make the screen material unique which will allow you to edit it. Second, let's tweak the screen material a bit so it's like the image below. You can remove the default screen graph and then add the following objects by right-clicking: `add/Vertex Inputs/Texcoord/Texcoord0`, `add/Vertex Inputs/Math/Multiply`, `add/Sampling/Sample Texture`, `add/Output/Unlit Base`. 

Important: Be sure to click on the Sample Texture node and rename its Sampler Slot Name to be "color_map". We'll reference this later to update the texture as we animate the gif frames.

![image](https://cloud.githubusercontent.com/assets/4054655/23757297/6293fed2-04b4-11e7-83f4-4c6982de08db.png)

# GIF Importer

If you remember, when we saved Giphy from the giphy viewer, the file explorer was opened on the folder containing the saved `gif` and all the PNG frames.

We were able to import PNG into our project since Stingray Editor supporting importing PNG by default, but it doesn't support importing GIFs.

So let's add an import extension to tell Stingray how to import GIF.

Create a new module named `giphy-importer.js` and add the following.

#### `giphy-importer.js`
```js
define(require => {
    'use strict';

    const _ = require('lodash');
    const stingray = require('stingray');
    const projectService = require('services/project-service');
    const engineService = require('services/engine-service');

    /**
     * Copy gif into project and compile.
     */
    function importGiphy (importOptions, previousResult, files, destination, flags) {
        if (!_.isArray(files))
            files = [files];

        console.log(importOptions, previousResult, files, destination, flags);

        return projectService.getCurrentProjectPath().then(function (projectPath) {
            let projectDestination = stingray.path.join(projectPath, destination);

            let importFile = sourceFilePath => {
                let fileName = stingray.path.basename(sourceFilePath, true);
                let fileDestination = stingray.path.join(projectDestination, fileName);
                return stingray.fs.copy(sourceFilePath, fileDestination);
            };

            return Promise.series(files.map(path => {
                if (stingray.fs.exists(path))
                    return path;
                return stingray.path.join(projectPath, path);
            }), importFile).then(() => engineService.enqueueDataCompile());
        });

    }

    return {
        importGiphy,
    };
});
```

So we have the import, but we also need to declare it's import extension in the plugin descriptor with:

```

// Add an GIF importer.
imports = [
    { types = ["gif"] label = "Giphy" do = [ { type = "js" module = "giphy-importer" function_name = "importGiphy" } ] }
]
```

Reload the editor views and drag an GIF from your file explorer. See now it works.

So try again, import this [gif](https://draoqiiudbn5q.cloudfront.net/hackathon/stingray.gif) in `giphy_resources`.

![gif](https://draoqiiudbn5q.cloudfront.net/hackathon/stingray.gif)
<https://draoqiiudbn5q.cloudfront.net/hackathon/stingray.gif>

But we do not see it in the asset browser by default?

[Don't panic!](https://www.amazon.ca/Dont-Panic-Douglas-Hitchhikers-Galaxy/dp/1848564961)

We'll add an asset type extension that will do just that:

```
// Add an asset type to see GIF assets in the asset browser.
asset_types = [
    { type = "gif" category = "Images" icon = "sample_project/thumbnail.png" }
]
```

# Engine GIF compiler

In order to see our Giphy in the engine we need to instruct the engine how to compile them.

Again, let's make sure our Visual Studio projects are up-to-date.

Since this will also depends on stb, we need to update the our CMake script `engine/CMakeLists.txt`:

```cmake

# Fetch stb as an external library.
include(ExternalProject)
set(STB_INSTALL_LOCATION "${PROJECT_BINARY_DIR}/external/stb")
set(STB_INSTALL_LOCATION_LIB_DIR "${STB_INSTALL_LOCATION}/lib")
list(APPEND CMAKE_ARGS "-D${CACHE_VAR}${CACHE_VAR_TYPE}=${${CACHE_VAR}}")
ExternalProject_Add(external_stb
    GIT_REPOSITORY https://github.com/jschmidt42/stb-cmake.git
    CMAKE_ARGS "${CMAKE_ARGS};-DCMAKE_INSTALL_PREFIX=${STB_INSTALL_LOCATION}"
    UPDATE_COMMAND ""
)

# ...
 
# Setup stb component
add_library(stb STATIC IMPORTED)
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_DEBUG "${STB_INSTALL_LOCATION_LIB_DIR}/debug/stb${LIB_SUFFIX}")
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_DEV "${STB_INSTALL_LOCATION_LIB_DIR}/dev/stb${LIB_SUFFIX}")
set_property(TARGET stb PROPERTY IMPORTED_LOCATION_RELEASE "${STB_INSTALL_LOCATION_LIB_DIR}/release/stb${LIB_SUFFIX}")
add_dependencies(stb external_stb)
add_dependencies(${PROJECT_NAME} stb)
include_directories(${STB_INSTALL_LOCATION}/include)
target_link_libraries(${PROJECT_NAME} stb)

```

 > ./make.rb --no-build
 
The solution should be generated under `build/engine/win64/engine_plugin.sln`
 
Open this up in Visual Studio.

Let's define the resource type

```cpp
// Data compiler resource properties
int RESOURCE_VERSION = 1;
const char *RESOURCE_EXTENSION = "gif";
const IdString64 RESOURCE_ID = IdString64(RESOURCE_EXTENSION);
```

And the compiler routine to save as is the GIF blob data.

```cpp
/**
 * Define plugin resource compiler.
 */
DataCompileResult gif_compiler(DataCompileParameters *input)
{
	auto source_data = data_compile_params->read(input);
	if (source_data.error)
		return source_data;
	return pack_source_data_with_size(input, source_data);
}
```

Let's add to `boot.package` package inclusions for `gif`:

```
gif = ["gifs/*", "giphy_resources/*"]
```

Got something to debug?

Open debugging options, then enter those settings:

**Command**: `<exe path of stingray_engine...exe>`, i.e. `G:\stingray\build\binaries\engine\win64\dev\stingray_win64_dev.exe`
**Command Arguments**: `--toolchain "G:\stingray\build\binaries" --plugin-dir "G:\giphy-plugin\plugin\binaries\engine\win64\debug"`

Next, we can always rename our plugin name in the source too:
 
 ```cpp
 /**
  * Returns the plugin name.
  */
 const char* get_name() { return "giphy_plugin"; }
 ```

Again, we'll need to use stb to load GIF frames from a memory buffer:

```cpp
/**
 * Load all GIF animations from a memory buffer.
 */
STBIDEF unsigned char *gif_load_frames(stbi_uc const *buffer, int len, int *x, int *y, int *frames)
{
	typedef struct gif_result_t {
		int delay;
		unsigned char *data;
		struct gif_result_t *next;
	} gif_result;

	stbi__context s;
	unsigned char *result;

	stbi__start_mem(&s, buffer, len);

	if (stbi__gif_test(&s)) {
		int c;
		stbi__gif g;
		gif_result head;
		gif_result *prev = nullptr, *gr = &head;

		memset(&g, 0, sizeof(g));
		memset(&head, 0, sizeof(head));

		*frames = 0;

		while ((gr->data = stbi__gif_load_next(&s, &g, &c, 4))) {
			if (gr->data == (unsigned char*)&s) {
				gr->data = nullptr;
				break;
			}

			if (prev) prev->next = gr;
			gr->delay = g.delay;
			prev = gr;
			gr = (gif_result*)stbi__malloc(sizeof(gif_result));
			memset(gr, 0, sizeof(gif_result));
			++(*frames);
		}

		STBI_FREE(g.out);

		if (gr != &head)
			STBI_FREE(gr);

		if (*frames > 0) {
			*x = g.w;
			*y = g.h;
		}

		result = head.data;

		if (*frames > 1) {
			unsigned int size = 4 * g.w * g.h;
			unsigned char *p;

			result = (unsigned char*)stbi__malloc(*frames * (size + 2));
			gr = &head;
			p = result;

			while (gr) {
				prev = gr;
				memcpy(p, gr->data, size);
				p += size;
				*p++ = gr->delay & 0xFF;
				*p++ = (gr->delay & 0xFF00) >> 8;
				gr = gr->next;

				STBI_FREE(prev->data);
				if (prev != &head) STBI_FREE(prev);
			}
		}
	} else {
		stbi__result_info result_info;
		result = (unsigned char*)stbi__load_main(&s, x, y, frames, 4, &result_info, 0);
		*frames = !!result;
	}

	return result;
}
```

And 

```cpp 
#define STB_IMAGE_IMPLEMENTATION
#define STB_IMAGE_WRITE_IMPLEMENTATION
#define STBI_ONLY_PNG
#define STBI_ONLY_GIF
 
#include <stb_image.h>
#include <stb_image_write.h>
```

In order to associate our Giphy with an unit, we'll add a simple structure to do book keeping:

```cpp
struct UnitGiphy
{
	// Used to reused released giphy slots
	bool used;

	// Used to find an existing giphy data.
	CApiUnit* unit_instance;

	// Gif image data
	unsigned width;
	unsigned height;
	unsigned char* gif_data;

	// Texture data
	unsigned texture_buffer_handle;

	// Playback data
	unsigned frame_count;
	unsigned current_frame;
	float next_frame_delay;
};

/**
 * Hold all loaded Giphies.
 */
Array<UnitGiphy>* giphies = nullptr;

```

Let's create our Giphy array in `setup_plugin`

```cpp
/**
 * Setup plugin runtime resources.
 */
void setup_plugin(GetApiFunction get_engine_api)
{
	//...

	giphies = MAKE_NEW(_allocator, Array<UnitGiphy>, _allocator);
}
```

And cleanup on shudown:

```cpp
/**
 * Release giphy data and mark slot as unused.
 */
void release_giphy(UnitGiphy& giphy)
{
	// Mark this slot as unused, so reusable.
	giphy.used = false;

	// Dispose of GIF animation image data.
	STBI_FREE(giphy.gif_data);

	// Release the texture buffer resource.
	render_buffer->destroy_buffer(giphy.texture_buffer_handle);
	giphy.texture_buffer_handle = INVALID_HANDLE;
}

/**
 * Release plugin resources.
 */
void shutdown_plugin()
{
	if (giphies) {
		for (unsigned i = 0; i < giphies->size(); ++i) {
			auto& ug = (*giphies)[i];
			if (!ug.used)
				continue;
			release_giphy(ug);
		}
		MAKE_DELETE(_allocator, giphies);
	}

	// ...
}
```

We can load GIF, map them, but when? Let's do that when units get spawned.
 Each time a unit get's spawned we'll check if it has a Giphy resource associated, if yes, then we'll
 load it up and bind them.

First let's tell the engine that we want to be notified about unit spawned and unspawned.

```cpp
extern "C" {

	/**
	 * Load and define plugin APIs.
	 */
	PLUGIN_DLLEXPORT void *get_plugin_api(unsigned api)
	{
		using namespace PLUGIN_NAMESPACE;

		if (api == PLUGIN_API_ID) {
			static PluginApi plugin_api = { nullptr };
			...
			plugin_api.units_spawned = units_spawned;
			plugin_api.units_unspawned = units_unspawned;
			...
			return &plugin_api;
		}
		return nullptr;
	}
}
```

Let's implement `units_spawned` and `units_unspawned`

```cpp
/**
 * Searches for a unit's giphy.
 */
UnitGiphy* find_giphy(CApiUnit* unit)
{
	for (unsigned g = 0; g < giphies->size(); ++g) {
		auto& ug = (*giphies)[g];
		if (ug.used && ug.unit_instance == unit)
			return &ug;
	}
	return nullptr;
}

/**
 * When new units spawn, we check if they have a giphy resource assigned and 
 * update their respective mesh material.
 */
void units_spawned(CApiUnit **units, unsigned count)
{
	#if _DEBUG
		#define LOG_AND_CONTINUE(msg, ...) { log->warning(RESOURCE_EXTENSION, error->eprintf(msg, ##__VA_ARGS__)); continue; }
	#else
		#define LOG_AND_CONTINUE(msg, ...) { continue;  }
	#endif
		
	for (unsigned i = 0; i < count; ++i) {
		auto unit_ref = unit->reference(units[i]);

		#if _DEBUG
			auto unit_resource_name = unit->unit_resource_name(units[i]);
		#endif

		// Define script data field name to get.
		const auto giphy_resource_indice = "giphy_resource";
		const auto mesh_index_indice = "giphy_mesh_index";
		const auto material_slot_name_indice = "giphy_material_slot_name";

		// Do not continue if this unit does not have any Giphy resource.
		if (!stingray::Data->Unit->has_data(unit_ref, 1, giphy_resource_indice))
			continue;
		
		// Make sure the unit has all the data we need to display a Giphy on it.
		if (stingray::Unit->num_meshes(unit_ref) == 0 ||
			!stingray::Data->Unit->has_data(unit_ref, 1, mesh_index_indice) || 
			!stingray::Data->Unit->has_data(unit_ref, 1, material_slot_name_indice))
			LOG_AND_CONTINUE("Unit #ID[%016llx] is missing Giphy property script data.", unit_resource_name);

		// Get script data values
		auto giphy_mesh_index = (unsigned)*(float*)stingray::Data->Unit->get_data(unit_ref, 1, mesh_index_indice).pointer;
		auto giphy_resource_name = (const char*)stingray::Data->Unit->get_data(unit_ref, 1, giphy_resource_indice).pointer;
		auto giphy_material_slot_name = (const char*)stingray::Data->Unit->get_data(unit_ref, 1, material_slot_name_indice).pointer;

		// We need a valid material slot name
		if (strlen(giphy_material_slot_name) == 0)
			LOG_AND_CONTINUE("Unit #ID[%016llx] has an invalid material slot name", unit_resource_name);

		// Make sure we can load the Giphy resource.
		if (!resource_manager->can_get(RESOURCE_EXTENSION, giphy_resource_name))
			LOG_AND_CONTINUE("Cannot get unit #ID[%016llx] giphy resource", unit_resource_name);

		// Get the unit mesh reference on which to display the Giphy.
		auto unit_mesh = stingray::Unit->mesh(unit_ref, giphy_mesh_index, nullptr);
		if (stingray::Mesh->num_materials(unit_mesh) == 0)
			LOG_AND_CONTINUE("Unit #ID[%016llx] has no material", unit_resource_name);

		// Get compiled GIF resource data and length.
		unsigned gif_data_len = 0;
		unsigned char* gif_resource_data = (unsigned char*)resource_manager->get(RESOURCE_EXTENSION, giphy_resource_name);
		memcpy(&gif_data_len, gif_resource_data, sizeof(gif_data_len));
		gif_resource_data += sizeof(gif_data_len);

		// Load GIF image data.
		int width = 0, height = 0, frames = 0;
		auto gif_frames_data = gif_load_frames((stbi_uc*)gif_resource_data, gif_data_len, &width, &height, &frames);
		if (gif_frames_data == nullptr)
			LOG_AND_CONTINUE("Cannot parse unit #ID[%016llx] giphy resource data", unit_resource_name);
		
		// Create texture buffer view
		RB_TextureBufferView texture_buffer_view;
		memset(&texture_buffer_view, 0, sizeof(texture_buffer_view));
		texture_buffer_view.width = width;
		texture_buffer_view.height = height;
		texture_buffer_view.depth = 1;
		texture_buffer_view.mip_levels = 1;
		texture_buffer_view.slices = 1;
		texture_buffer_view.type = RB_TEXTURE_TYPE_2D;
		texture_buffer_view.format = render_buffer->format(RB_INTEGER_COMPONENT, false, true, 8, 8, 8, 8); // ImageFormat::PF_R8G8B8A8;

		// Create and initialize texture buffer with first GIF frame.
		auto frame_size = width * height * 4;
		auto texture_buffer_handle = render_buffer->create_buffer(frame_size, RB_VALIDITY_UPDATABLE, RB_TEXTURE_BUFFER_VIEW, &texture_buffer_view, gif_frames_data);
		auto texture_buffer = render_buffer->lookup_resource(texture_buffer_handle);

		// Update the mesh material with the newly created texture buffer resource.
		auto mesh_mat = stingray::Mesh->material(unit_mesh, 0);
		auto material_slot_id = IdString32(giphy_material_slot_name).id();
		stingray::Material->set_resource(mesh_mat, material_slot_id, texture_buffer);

		// Associate and track the Giphy data for this unit.
		UnitGiphy ug;
		ug.used = true;
		ug.unit_instance = units[i];
		ug.gif_data = gif_frames_data;
		ug.width = width;
		ug.height = height;
		ug.texture_buffer_handle = texture_buffer_handle;

		// Initialize playback data.
		auto delay = (stbi__uint16)*(gif_frames_data + frame_size);
		ug.current_frame = 0;
		ug.frame_count = frames;
		ug.next_frame_delay = delay / 100.0f;

		// Find an unused giphy slot.
		bool reused = false;
		for (unsigned g = 0; g < giphies->size(); ++g) {
			if ((*giphies)[g].used)
				continue;
			(*giphies)[g] = ug;
			reused = true;
			break;
		}
		if (!reused)
			giphies->push_back(ug);
	}
}

/**
 * When units gets unspawned, lets if we have a associated giphy, if yes, 
 * lets release it.
 */
void units_unspawned(CApiUnit **units, unsigned count)
{
	for (unsigned i = 0; i < count; ++i) {
		auto unit = units[i];
		auto unit_giphy = find_giphy(unit);
		if (unit_giphy)
			release_giphy(*unit_giphy);
	}
}
```

## **At this point if we test level, we should see a fixed frame. Close the editor, rebuild the plugin DLL with:**

> ./make.rb

## Engine GIF playback

Let's those Giphy become alive! To do so, we'll implement in `update_plugin` a playback loop for GIFs.

```cpp
/**
 * Each frame, playback the GIF animation.
 */
void update_plugin(float dt)
{
	for (unsigned g = 0; g < giphies->size(); ++g) {
		auto& ug = (*giphies)[g];

		// Skip unit that do not have a Giphy
		if (!ug.used)
			continue;

		// Update frame delay
		ug.next_frame_delay -= dt;

		// Play next frame if the delay was reached.
		if (ug.next_frame_delay <= 0.0f) {
			ug.current_frame = (ug.current_frame + 1) % ug.frame_count;
			const auto frame_size = ug.width * ug.height * 4;
			auto next_frame_data = ug.gif_data + ug.current_frame * (frame_size + 2); // + 2 for delay info
			render_buffer->update_buffer(ug.texture_buffer_handle, frame_size, next_frame_data);

			auto delay = (stbi__uint16)*(next_frame_data + frame_size);
			ug.next_frame_delay = delay / 100.0f;
		}
	}
}
```

# Resource and Project template extensions

At this point we should be able to download as many Giphy as we want, compile them and animate them in the engine.

## Resource extension

To wrap our work, let's define a resource extension that will share the `giphy_resources` for all project that
loads the Giphy plugin.

To do so, move `plugin/sample_project/giphy_resources` to `plugin/giphy_resources`.

Add a security token file in this folder named `.stingray-asset-server-directory`. It can be empty.

> touch .stingray-asset-server-directory

In the plugin descriptor, add the resource extension by defining:

```
// Map plugin resources
resources = [ { path = "giphy_resources" } ]
```

From now on, our Giphy TV will be available for all project. The folder is now a mapped folder and can be shown in the asset browser by enabling `Show mapped folders`.

![image](https://cloud.githubusercontent.com/assets/4054655/23757323/7766768c-04b4-11e7-9707-046542c5bf5f.png)

## Project extension

So our sample project is now ready for production. Define a project template extension in the plugin descriptor:

```
// Define a project template extension that will show up in the project manager.
templates = [
    { type = "project-template" path = "sample_project/sample.stingray_project" }
]
```

We should now be able to create new instances of our sample project from the project manager.

![image](https://cloud.githubusercontent.com/assets/4054655/23757356/96313174-04b4-11e7-98ad-3531a32cce66.png)

# Build It!

Let's do a final build:

> ./make.rb

> SUCCESSFULLY COMPLETED!

# Screenshots

## Project template to create many Giphy sample projects
![image](https://user-images.githubusercontent.com/4054655/29778316-f3064f24-8bdc-11e7-87b8-781d854b8f23.png)

## Giphy Viewer
![image](https://user-images.githubusercontent.com/4054655/29778480-88ae599a-8bdd-11e7-97d1-73e4dc86a7a8.png)

## Engine runtime playing gifs
![image](https://user-images.githubusercontent.com/4054655/29778364-1fdf54be-8bdd-11e7-99ba-e8b15f053927.png)

# What have we seen?
- [ ] Plugin descriptor (.plugin)
- [ ] Build system included.
	- [ ] The build system uses some ruby scripts (`make.rb`, `spm.rb`, etc.) and cmake.
	- [ ] The build system can handle external dependencies used by your extensions (e.g. [stb](https://github.com/nothings/stb) through [stb-cmake](https://github.com/jschmidt42/stb-cmake))
- [ ] Various engine and editor components, such as:
	- [ ] Engine resources 
		- [ ] Runtime resources, i.e. `plugin/giphy_resources`
		- [ ] These resources gets mapped at runtime by the editor.
	- [ ] Engine native plugin extension(s) (dll)
		- [ ] The editor loads and unloads these extensions at runtime using `runtime_libraries` extensions.
	- [ ] Editor native plugin extension(s) (dll)
	- [ ] Editor scripts/modules
	- [ ] Plugin extensions
		- [ ] `views`
		- [ ] `menus`
		- [ ] `asset_types`
		- [ ] `templates`
		- [ ] `resources`
		- [ ] `imports`
		- [ ] `runtime_libraries`
		- [ ] [but there's more...](http://help.autodesk.com/view/Stingray/ENU/?guid=__sdk_help_extend_editor_plugin_extensions_html)
- [ ] Bundle a sample project.

# Ship It!

You thought we were done? Almost!

Let's ship that plugin to the world!

As I mentioned at the beginning, the `plugin` folder is self-contained. you can zip it and share it.

So let's zip that folder, rename the zip file if wanted.

## Upload the plugin on SPM

https://gamedev.autodesk.com/stingray/plugins

Thanks,
