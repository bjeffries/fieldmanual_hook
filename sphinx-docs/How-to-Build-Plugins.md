# How to Build Plugins

Building your own plugin allows you to add custom functionality to Caldera. 

A plugin can be nearly anything, from a RAT/agent (like Sandcat) to a new GUI or a collection of abilities that you want to keep in "closed-source". 

Plugins are stored in the plugins directory. If a plugin is also listed in the local.yml file, it will be loaded into Caldera each time the server starts. A plugin is loaded through its hook.py file, which is "hooked" into the core system via the server.py (main) module.

> When constructing your own plugins, you should avoid importing modules from the core code base, as these can change. 
> There are two exceptions to this rule
> 1. The services dict() passed to each plugin can be used freely. Only utilize the public functions on these services 
> however. These functions will be defined on the services' corresponding interface.
> 2. Any c_object that implements the FirstClassObjectInterface. Only call the functions on this interface, as the others
> are subject to changing.

This guide is useful as it covers how to create a simple plugin from scratch. 
However, if this is old news to you and you're looking for an even faster start, 
consider trying out [Skeleton](https://github.com/mitre/skeleton)
(a plugin for building other plugins). 
Skeleton will generate a new plugin directory that contains all the standard
boilerplate. 

## Creating the structure

Start by creating a new directory called "abilities" in Caldera's plugins directory. In this directory, create a hook.py file and ensure it looks like this:
```python
name = 'Abilities'
description = 'A sample plugin for demonstration purposes'
address = None


async def enable(services):
    pass
```

The name should always be a single word, the description a phrase, and the address should be None, unless your plugin exposes new GUI pages. Our example plugin will be called "abilities".

## The _enable_ function

The enable function is what gets hooked into Caldera at boot time. This function accepts one parameter:

1. **services**: a list of core services that Caldera creates at boot time, which allow you to interact with the core system in a safe manner. 

Core services can be found in the app/services directory.

## Writing the code

Now it's time to fill in your own enable function. Let's start by appending a new REST API endpoint to the server. When this endpoint is hit, we will direct the request to a new class (AbilityFetcher) and function (get_abilities). The full hook.py file now looks like:
```python
from aiohttp import web

name = 'Abilities'
description = 'A sample plugin for demonstration purposes'
address = None


async def enable(services):
    app = services.get('app_svc').application
    fetcher = AbilityFetcher(services)
    app.router.add_route('*', '/get/abilities', fetcher.get_abilities)


class AbilityFetcher:

    def __init__(self, services):
        self.services = services

    async def get_abilities(self, request):
        abilities = await self.services.get('data_svc').locate('abilities')
        return web.json_response(dict(abilities=[a.display for a in abilities]))
```

Now that our initialize function is filled in, let's add the plugin to the default.yml file and restart Caldera. Once running, in a browser or via cURL, navigate to 127.0.0.1:8888/get/abilities. If all worked, you should get a JSON response back, with all the abilities within Caldera. 

## Making it visual

Now we have a usable plugin, but we want to make it more visually appealing. 

Start by creating a "templates" directory inside your plugin directory (abilities). Inside the templates directory, create a new file called abilities.html. Ensure the content looks like:
```html
<div id="abilities-new-section" class="section-profile">
    <div class="row">
        <div class="topleft duk-icon"><img onclick="removeSection('abilities-new-section')" src="/gui/img/x.png"></div>
        <div class="column section-border" style="flex:25%;text-align:left;padding:15px;">
            <h1 style="font-size:70px;margin-top:-20px;">Abilities</h1>
        </div>
        <div class="column" style="flex:75%;padding:15px;text-align: left">
            <div>
                {% for a in abilities %}
                    <pre style="color:grey">{{ a }}</pre>
                    <hr>
                {% endfor %}
            </div>
        </div>
    </div>
</div>
```

Then, back in your hook.py file, let's fill in the address variable and ensure we return the new abilities.html page when a user requests 127.0.0.1/get/abilities. Here is the full hook.py:

```python
from aiohttp_jinja2 import template, web

from app.service.auth_svc import check_authorization

name = 'Abilities'
description = 'A sample plugin for demonstration purposes'
address = '/plugin/abilities/gui'


async def enable(services):
    app = services.get('app_svc').application
    fetcher = AbilityFetcher(services)
    app.router.add_route('*', '/plugin/abilities/gui', fetcher.splash)
    app.router.add_route('GET', '/get/abilities', fetcher.get_abilities)


class AbilityFetcher:
    def __init__(self, services):
        self.services = services
        self.auth_svc = services.get('auth_svc')

    async def get_abilities(self, request):
        abilities = await self.services.get('data_svc').locate('abilities')
        return web.json_response(dict(abilities=[a.display for a in abilities]))

    @check_authorization
    @template('abilities.html')
    async def splash(self, request):
        abilities = await self.services.get('data_svc').locate('abilities')
        return(dict(abilities=[a.display for a in abilities]))

```
Restart Caldera and navigate to the home page. Be sure to run ```server.py```
with the ```--fresh``` flag to flush the previous object store database. 

You should see a new "abilities" tab at the top, clicking on this should navigate you to the new abilities.html page you created. 

## Adding documentation

Any Markdown or reStructured text in the plugin's `docs/` directory will appear in the documentation generated by the fieldmanual plugin. Any resources, such as images and videos, will be added as well.

## Caldera Plugin Hooks

Caldera provides plugins the ability to hook into the runtime when a link is added to any operation. This is facilated through a dictionary object in each executor ```executor.HOOKS```. The values in this dictionary contain function pointers to be called before the server queues the ability for execution.

To leverage this capability, plugins need to add their function pointer to the hook dictionary object. An example of how this can be accomplished is described below.

1. Modify your plugin hook.py to await a new service function that will add our executor hooks. This is done in the expansion method as the abilities files have been loaded by this point during server startup.
``` python
#hook.py
async def expansion(services):
    await services.get('myplugin_svc').initialize_code_hook_functions()
```
2. Update your plugin service script (e.g., myplugin_svc.py) to parse ability files and their executors. Add logic to hook into the executor's you are interested in modifying.
``` python
#myplugin_svc.py
async def initialize_code_hook_functions(self):
        self.log.debug('searching for abilities')
        ab_count = 0
        ex_count = 0
        hooks = 0
        for ability in await self.data_svc.locate('abilities'):
            ab_count += 1
            for executor in ability.executors:
                ex_count += 1
                """add your logic here"""
                if ability.plugin == "myplugin":
                    self.log.debug(f'{ability.ability_id} is being hooked.')
                    hooks +=1
                    """Make the key unique to your plugin"""
                    executor.HOOKS['myuniquekey'] = self.myplugin_hook
        self.log.debug(f'parsed {ab_count} abilities, {ex_count} executors, {hooks} hooks added.')

async def myplugin_hook(self, ability, executor):
    try:
        """Change the command"""
        executor.command = "my new command"

        """Adding a new payload"""
        executor.payloads.append("/tmp/mynewpayload.exe")

    except Exception as e:
        self.log.debug(f'Error while performing hook function: {e}')

```
In the example above, we hook each caldera ability with our unique plugin function that meets the following condition: 
``` python
#myplugin_svc.py -> see add your logic here
ability.plugin == "myplugin"
```
Consequently, the ability yaml file we are targeting would need to have the plugin defined as "myplugin"
```yaml
# 1811b7f2-3a73-11eb-adc1-0242ac120102.yml
- id: 1811b7f2-3a73-11eb-adc1-0242ac120102
  name: my awesome ability
  plugin: myplugin
```
You can use a myriad of criteria to determine which abilities or specific executors you are hooking into. In the example above we use the plugin name, but you could just as easily use the ability id, or a custom symbol you add to the ability or executor 'additional_info'. See below.

``` python
#myplugin_svc.py -> see add your logic here
ability.additional_info['hook'] == "myspecialhook" 
```

```yaml
# 1811b7f2-3a73-11eb-adc1-0242ac120103.yml
- id: 1811b7f2-3a73-11eb-adc1-0242ac120103
  name: my awesome ability
  plugin: myplugin
  hook: myspecialhook
```
