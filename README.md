# Fastify Svelte Partial Hydration

This plugin will help you add partially hydrated pages
to your `fastify-svelte` application.

## Table of contents

1. [Installation](#installation)
2. [Usage](#usage)

## Installation

To install this plugin, run the command:

```bash
# npm
npm install --save @elo7/fastify-svelte-partial-hydration
# yarn
yarn add @elo7/fastify-svelte-partial-hydration
```

## Usage
To use this plugin, we have to pass through 4 steps: to register, to make
the hydratable components globally callable, to add the hydration script
and to use the hydration component.

#### Steps
1. [Register](#register)
2. [Register hydratable-components](#register-hydratable-components)
3. [Hydration script](#hydration-ccript)
4. [Hydration component](#hydration-component)

#### Register

The first part is to register the plugin with the fastify-svelte, by passing
the default export of the plugin to the `plugins` array for the `fastify-svelte`:

```javascript
import sveltePartialHydration from
'@elo7/fastify-svelte-parital-hydration`;

app.register(sveltePlugin, {
	plugins: [sveltePartialHydration],
});
```

#### Register hydratable components

The second part is divided by two steps: generate scripts that will make
the components globally available and the register these scripts on your
page.

The hydratable components need to generate scripts that make them globally
available, below, you will see an example on how to do that using the `window`
object and the virtual module plugin for Rollup. Note that the generated
scripts are placed under

`/static/js/mobile/components/<component-name>/template.js`

```javascript
const componentsPath = 'src/views/mobile/components/';
const hydrateTemplates = glob.sync(`${componentsPath}/**/template.svelte`);

const clientSideConfig = (template) => {
	const { component } = template.match(/.+\/mobile\/.+\/(?<component>.+)\/template.svelte/).groups;

	return {
		input: component,
		output: {
			file: template.replace('src', 'static/js').replace('svelte', 'js'),
			format: 'iife',
		},
		plugins: [
			virtual({
				[component]: `
					import ${component} from './${template}';
					window[${component}] = ${component};
				`,
			}),
			svelte({
				css: false,
				dev,
				hydratable: true,
			}),
			resolve(),
			commonjs(),
		],
	};
};

export default [
	...hydrateTemplates.map(clientSideConfig);
];
```

This plugins makes the key `componentsToHydrate` available, which will contain
all components's names that should be hydrated. In this step, we are calling
the scripts generated by the rollup configuration above:

```javascript
// template.js
export default ({ head, css, componentsToHydrate, content }) => `
// ...
<body>
	// ...

	${componentsToHydrate.map(component => `<script async src='/static/js/views/mobile/components/${component}/template.js'></script>`)}
</body>
`;
```

#### Hydration script

The third part is to create your component hydrator, which is a
function that will receive the name of the component, the props and the
DOM's element where the component will be mounted, and call the Svelte
component constructor:

```javascript
import hydrate from '@elo7/fastify-svelte-partial-hydration/hydrate';

const componentBuilder = ({ component, element, props }) => {
	new window[component]({
		hydrate: true,
		props: JSON.parse(props),
		target: element,
	});
};

hydrate(componentBuilder);
```

Register the script above on your `template.js`;

```javascript
// template.js
export default (...) => `
// ...
<body>
	// ...
	<script async src='/static/js/hydrate.js'></script>
</body>
`;
```

**NOTE**: The code above assumes that the client side compilation of your
svelte plugins are stored within the `window` object, this can be achieved
by using virtual modules with rollup. There is an example on how to do this
in the [Register hydratable components](#register-hydratable-components) section.

#### Hydration component

The fourth and final part is using the Hydrate Svelte component, you need to pass
the name of the component, the component itself and the props object that will
be passed to the components.

```javascript
import Hydrate from '@elo7/fastify-svelte-partial-hydration/Hydrate.svelte';
import SearchBar from './SearchBar.svelte';

<Hydrate
	component={SearchBar}
	name="SearchBar"
	props={{
		autocomplete: true,
	}}
/>
```
