# Interactive Components in Classic Plone with Svelte

**Talk Plone Conference 2020  
Katja Süss**

[https://interactive-components-in-classic-plone.readthedocs.io](https://interactive-components-in-classic-plone.readthedocs.io)

We will look at two examples: 

* search for experts via competence, name and filter by region
* search for locations via map.

What you will learn:

* Set up a Plone package with Plone customizations and JS code and everything else in one.

Maik Derstappen created a handy bob template that creates a Plone package with all essential elements for a Svelte app integrated in Plone.
plonecli create addon rohberg.expertsearch

    cd rohberg.expertsearch
    plonecli add svelte_app

    cd svelte_src/my_svelte_app 
    npm install
    npm run dev

Your Svelte app opens in browser http://localhost:10001/

Open your package in editor and change the file ‘App.svelte’. Add a `<h2>Hello world!</h2>`
and see the change reflected in browser.

Change svelte_src/my-svelte-app/src/main.js:

    let targets = document.getElementsByClassName("my-svelte-app");

Change src/rohberg/expertsearch/svelte_apps/my-svelte-app/index.html: 

    <div class="my-svelte-app"></div>

We will now see how this Svelte app is integrated in Plone.

Open another terminal window, go to the package folder and build your Plone.

    cd ../..  
    plonecli build 

run the newly created Zope instance

    ./bin/instance fg

![Screenshot](img/personsearch_install.png)

Create Plone instance and install the Plone package on “Site Setup”.

Integrate your Svelte app in home page by pasting in TinyMCE

    <div class="my-svelte-app"></div> 

![Svelte First View](img/svelte_first_view.png)

The integration is done.

## Person Search

- search form
- fetch membrane data 
- display fetched membrane users data

We start with a search form.
Note: If you use VSCode, install `svelte.svelte-vscode` to help you with the code by highlighting and code tips.

For a simple form with search field and selection of the region paste the following code in a new file ExpertSearch.svelte’

``` svelte hl_lines="12 18"
<script>
  let searchstring = '';
  let region = 'all regions';
  let menuregions = [
    'all regions',
    'Zürich Stadt',
    'Zürich Oberland',
    'Winterthur'
  ]
  
  const handleClickRegion = (event) => {
    region = event.target.value;
    console.log(searchstring, region);
  }
</script>

<h2>Expert Search</h2>
<form action="">
  <input class="searchstring" bind:value={searchstring} placeholder="search">
  <br>
  {#each menuregions as region}
    <input 
      type=button
      class="regionbutton"
      on:click|preventDefault={handleClickRegion}
      value={region}>
  {/each}
</form>
<p><i>Search{#if searchstring}{' '}for {searchstring}{/if} in {region}</i></p>
```

Add component `ExpertSearch` to your `App.svelte`.

    <script>  
      let name = "my-svelte-app";

      import ExpertSearch from './ExpertSearch.svelte';
    </script>


    <main>
      <ExpertSearch />
    </main>

You see a simple form that even has an event handler.  
The info line below reflects the input of searchstring, because the input value is bound to variable searchstring.  
It also reflects the selection of region, because the event handler sets the value of the variable region.  
The loop `#each` displays the buttons to filter the results to region.  

For theming see package `rohberg.expertsearch` [^1].


![Personsearch values](img/personsearch_values.png)

We can use the values of searchstring and region to fetch the according data of matching membrane users.


We need some experts with information about their competence, region and organization.

Let’s add the Membrane dependency to our Plone package.

    install_requires=[
        'setuptools',
        # -*- Extra requirements: -*-
        'z3c.jbot',
        'plone.api>=1.8.4',
        'plone.restapi',
        'plone.app.dexterity',
        'Products.membrane>= 5.0.0a1',
        'dexterity.membrane>=3.0.0a1',
    ],

We create a behavior 'expert' to add the fields competence, region and organization.

Run 

    plonecli add behavior

Customize the schema:

    @provider(IFormFieldProvider)
    class IExpert(model.Schema):

        competence = schema.TextLine(
            title=_(u'Competence'),
            required=True,
        )
        region = schema.TextLine(
            title=_(u'Region'),
            required=True
        )
        organisation = schema.TextLine(
            title=_(u'Organisation'),
            required=True,
        )

Let’s now show some search results.


## Fetching the data from Plone via RestAPI

We want to distinguish between Svelte standalone and in Plone:

    npm install @rollup/plugin-replace --save-dev

`rollup.config.js`:

    import replace from '@rollup/plugin-replace';

    const production = !process.env.ROLLUP_WATCH;

    export default {
      plugins: [
        replace({
          // two level deep object should be stringified
          process: JSON.stringify({
            env: {
              isProd: production,
            }
          }),
        }),
      ],
    };

Now we can define the API URL to be `/` for production (integration in Plone) and `localhost:8080/Plone/` for developing with a standalone Svelte app.

So we have

    let apiURL = process.env.isProd ? '/' : 'http://localhost:8080/Plone/';
    apiURL = apiURL + '@search?portal_type=dexterity.membrane.member&fullobjects=1'
    let experts = [];

and will fetch the data of experts on mount.

We define an asynchronous function that gets called on mount of the component, on click on a region and on input of a searchstring.

``` svelte hl_lines="2 22-43"
<script>
  import { onMount } from "svelte";
  import { scale } from "svelte/transition";
  import { flip } from "svelte/animate";

  let searchstring = '';
  let region = 'all regions';
  // TODO menuregions according values in backend index
  let menuregions = [
    'all regions',
    'Zürich Stadt',
    'Zürich Oberland',
    'Winterthur'
  ]
  
  let apiURL = process.env.isProd ? '/' : 'http://localhost:8080/Plone/';
  apiURL = apiURL + '@search?portal_type=dexterity.membrane.member&fullobjects=1&sort_on=last_name&sort_order=ascending';

  // TODO create store for experts
  let experts = [];

  async function getExperts(url=apiURL) {
    fetch(url, {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json"
      }
    })
    .then(response => {
      if (!response.ok) {
        throw new Error('Network response was not ok');
      }
      return response.json();
    })
    .then(data => {
      experts = data?.items || [];
      return experts;
    })
    .catch(error => {
      console.error('There has been a problem with your fetch operation:', error);
    });
  };

  onMount(() => {
    getExperts()
  }) 

  const handleClickRegion = (event) => {
    region = event.target.value;
    getExperts((region == 'all regions') ? apiURL : apiURL + '&region=' + encodeURI(region));
  }
</script>
```

The fetch is addressing the REST API of Plone, its sending a GET request with json data. 
We save the expert data to variable experts and catch errors like network errors.


**Note**

The Fetch API provides a JavaScript interface for accessing and manipulating parts of the HTTP pipeline, such as requests and responses. It also provides a global fetch() method that provides an easy, logical way to fetch resources asynchronously across the network.

[https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)

To have some information for empty search result, we add an else section to the each section.

``` svelte hl_lines="12-13"
<div class="cards">
  {#each experts as expert, i (expert['@id'])}
    <div class="card" transition:scale animate:flip={{ duration: 300 }}>
      <span class="fullname">{expert.first_name} {expert.last_name}</span>
      <br>
      <span class="competence">{expert.competence}</span>
      <br>
      <span class="organisation">{expert.organisation}</span>
      <br>
      <span class="region">{expert.region}</span>
    </div>
  {:else}
    <p>no experts found</p>
  {/each}
</div>
```

You see a network error in console. Why is this? Please go to your Plone configuration panel and install `plone.restapi`.

To solve the CORS situation we modify the buildout. See [https://github.com/plone/plone.rest#cors](https://github.com/plone/plone.rest#cors) for more information

`buildout.cfg`:

    [instance]
    zcml-additional =
      <configure xmlns="http://namespaces.zope.org/zope"
                xmlns:plone="http://namespaces.plone.org/plone">
      <plone:CORSPolicy
        allow_origin="http://localhost:10001,http://127.0.0.1:10001"
        allow_methods="DELETE,GET,OPTIONS,PATCH,POST,PUT"
        allow_credentials="true"
        expose_headers="Content-Length,X-My-Header"
        allow_headers="Accept,Authorization,Content-Type,X-Custom-Header,Origin"
        max_age="3600"
        />
      </configure>

Run buildout.


You see all Persons.  
For **filtering** the search by region or **searching** for name, competence and organisation we will define an index for region and make name and competence searchable.

`catalog.xml`

``` xml hl_lines="3-5"
    <?xml version="1.0"?>
    <object name="portal_catalog">
      <index name="region" meta_type="FieldIndex">
        <indexed_attr value="region"/>
      </index>
    </object>
```

TODO name, organisation and competence searchable

Customize the schema of the bahavior:

    @provider(IFormFieldProvider)
    class IExpert(model.Schema):
        """
        """

        dexteritytextindexer.searchable('competence')
        # dexteritytextindexer.searchable('region') # extra index for filtering by region
        dexteritytextindexer.searchable('organisation')
        
Add behavior `collective.dexteritytextindexer.behavior.IDexterityTextIndexer` to make Membrane User searchable by competence, organisation, first_name and last_name.

Create and customize `profiles/default/types/dexterity.membrane.member.xml`by 

    <!-- enabled behaviors -->
    <property name="behaviors">
      <element value="rohberg.expertsearch.expert" />
      <element value="collective.dexteritytextindexer.behavior.IDexterityTextIndexer" />
    </property>

With this in place, membrane users are searchable by competence and organisation fields of the new expert behavior.   
We still need the first_name and last_name to be searchable. So add the following to a new file `indexer.py`:

from collective import dexteritytextindexer
from dexterity.membrane.content.member import IMember
from zope.component import adapts
from zope.interface import implementer

    @implementer(dexteritytextindexer.IDynamicTextIndexExtender)
    class MembraneSearchableTextExtender(object):
        adapts(IMember)    

        def __init__(self, context):
            self.context = context

        def __call__(self):
            """Extend the searchable text with a custom string"""
            return "{} {}".format(self.context.last_name, self.context.first_name)

Register this indexer in `configure.zcml`

    <adapter factory=".indexers.MembraneSearchableTextExtender"
          name="MembraneSearchableTextExtender"
          />

At last we add an index for last_name to sort the search results for last_name.

    <object name="portal_catalog">
      <!--<column value="my_meta_column"/>-->  
      <index name="region" meta_type="FieldIndex">
        <indexed_attr value="region"/>
        <indexed_attr value="last_name"/>
      </index>
    </object>

Restart Plone and reinstall your Plone add-on. If you already have Membrane users recatalog.

Check in Plone Backend that you find your experts by competence, last name etc..

Back to Svelte.

We made the interesting information searchable. Let's bind TODO














## Information about Svelte
* Official Site with tutorial and API info: [https://svelte.dev/](https://svelte.dev/)
* Excellent training: Maximilian Schwarzmüller on Udemy [https://www.udemy.com/course/sveltejs-the-complete-guide/learn/lecture/14689784#overview](https://www.udemy.com/course/sveltejs-the-complete-guide/learn/lecture/14689784#overview)
* Mozilla Svelte tutorials [https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks#Svelte_tutorials](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks#Svelte_tutorials)


## Example Packages

[^1]: [rohberg.expertsearch](https://github.com/rohberg/rohberg.expertsearch)