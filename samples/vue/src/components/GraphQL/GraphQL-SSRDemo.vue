<!--
-->
<template>
  <div data-e2e-id="graphql-ssr">
    <h2>GraphQL Server Side Rendering Demo</h2>

    <p>
      Server Side GraphQL executes GraphQL queries directly against the Sitecore GraphQL endpoint,
      prior to rendering the app to a string. This sample app uses the <code>vue-apollo</code> library to
      manage GraphQL queries.
    </p>
    <p>
      <strong>NOTE:</strong> when using the <code>vue-apollo</code> library prefetch option,
      GraphQL queries are executed <em>prior</em> to app rendering, so queries do not have access to
      the component instance, i.e. <code>this</code>.
      Unfortunately, that means no "easy" access to rendering data for the component.
      You do have access to the full SSR layout service data via the <code>state</code> object that JSS passes in to
      the query during SSR, but you'll need to resolve any data from that object manually. Also important to
      note that this is not a limitation of Sitecore JSS but rather the <code>vue-apollo</code> library. Ideally,
      that library would implement a <code>renderToStringWithData</code> method similar to <code>react-apollo</code>
      that would allow queries to execute with instantiated components.
    </p>
    <p>
      Expected behavior for this component:
      <ul>
        <li>
          <strong>Connected Mode:</strong> the GraphQL query will execute in the browser after component load.
          The <code>prefetch</code> option will not execute.
        </li>
        <li>
          <strong>Integrated Mode:</strong> the <code>prefetch</code> option of the GraphQL query will execute on the server
          and output will be rendered in the markup generated by the app on the server. After the initial load / render from
          the server, you can manage the query like a client-side query.
        </li>
        <li><strong>Disconnected Mode:</strong> GraphQL requires connected mode, headless
        connected mode, or integrated mode to work.</li>
      </ul>
    </p>

    <p v-if="loadingQueriesCount > 0" class="alert alert-info">GraphQL query is executing...</p>
    <p v-if="error" class="alert alert-danger">GraphQL query error: {{ error.toString() }}</p>
    <div v-if="contextItem">
      <h4>Route Item (via SSR Connected GraphQL)</h4>
      id: {{ contextItem.id }}
      <br />
      page title: {{ contextItem.pageTitle.value }}
      <br />
      children:
      <ul>
        <li v-for="(child) in contextItem.children" :key="child.id">
          <router-link :to="child.url">{{child.pageTitle.value}}</router-link>&nbsp; (editable title too! <sc-text :field="child.pageTitle.jss" />)
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import { ConnectedDemoQuery } from './GraphQL-ConnectedDemo.query.graphql';

const Text = () => import('@sitecore-jss/sitecore-jss-vue').then((m) => m.Text);

export default {
  name: 'GraphQL-SSRDemo',
  props: {
    fields: {
      type: Object,
    },
    rendering: {
      type: Object,
    },
  },
  components: {
    ScText: Text,
  },
  data() {
    return {
      loadingQueriesCount: 0,
      error: null,
    };
  },
  computed: {
    datasource() {
      return this.queryData && this.queryData.datasource;
    },
    contextItem() {
      return this.queryData && this.queryData.contextItem;
    },
  },
  apollo: {
    queryData: {
      query: ConnectedDemoQuery,
      variables() {
        const defaultValue = '{00000000-0000-0000-0000-000000000000}';
        const variables = {
          contextItem: this.$jss ? this.$jss.sitecoreContext().itemId : defaultValue,
          datasource: defaultValue,
        };

        if (!variables.contextItem) variables.contextItem = defaultValue;

        return variables;
      },
      error(error) {
        this.error = error;
      },
      loadingKey: 'loadingQueriesCount',
      update(data) {
        // By default, vue-apollo will try to add a property to the component instance
        // using the query key specified above, e.g. `queryData`.
        // Also by default, vue-apollo will try to extract data from the query result using
        // that same query key, e.g. result.data.queryData.
        // However, the demo query we use returns multiple (2) fields in the result data: `datasource` and `contextItem`.
        // Therefore, we need to use the `update` function to assign the result data object
        // to the component data property.
        // The end result is that you use `this.queryData.contextItem` or `this.queryData.datasource`
        // to access the query result data.
        return data;
      },
      // NOTE: prefetch is called prior to app rendering, so it does not have access to
      // the component instance, i.e. `this`.
      // Unfortunately, that means no "easy" access to rendering data for the component.
      // You do have access to the full SSR data via the `state` object, but you'll need to
      // resolve the rendering data manually.

      // Also, it is imperative that the variables data returned by the prefetch function
      // are identical to the variables data returned by the `variables()` property.
      // If the variables don't match, the query won't be able to retrieve cached data during SSR.
      // eslint-disable-next-line no-unused-vars
      prefetch: ({ route, state }) => {
        return {
          contextItem:
            (state && state.sitecore && state.sitecore.route && state.sitecore.route.itemId) ||
            '{00000000-0000-0000-0000-000000000000}',
          datasource: '{00000000-0000-0000-0000-000000000000}',
        };
      },
    },
  },
};
</script>
