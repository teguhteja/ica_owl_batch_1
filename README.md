### Odoo Owl Tutorial: Movie Partner Client Action

First, Odoo Owl tutorial guides you through creating a custom movie partner client action in Odoo. Moreover, this Odoo Owl tutorial shows how to use useState, onWillStart, and useRef hooks. Additionally, you will explore code snippets with clear commentary. Furthermore, you can visit the [Odoo OWL docs](https://www.odoo.com/documentation/16.0/developer/reference/frontend/owl.html) for extra reference.

## Understanding the ICA Movie OWL Component Action

First, this section explains the role of the ICA Movie component in Odoo OWL. Furthermore, the component registers itself in the action registry so that Odoo can load it on demand. Additionally, it initializes reactive state and DOM references in the setup method. Moreover, it fetches partner records before rendering to give users immediate data. Finally, this overview sets the stage before we dive deep into each file.

### What Is an OWL Component in Odoo?

First, Odoo Web Library (OWL) provides a reactive, component‑based UI framework. Furthermore, OWL uses life‑cycle hooks like `onWillStart` to run code before rendering. Additionally, `useState` creates reactive state objects, and `useRef` links the component logic to specific DOM elements. Moreover, OWL ties seamlessly to Odoo’s RPC layer through the `env.services.orm` service. Finally, grasping OWL components helps you craft dynamic modules that scale.

#### Key OWL Hooks and APIs

First, the `useState` hook tracks data changes and triggers re‑rendering. Furthermore, `onWillStart` runs asynchronous code before the first render. Additionally, `useRef` gives direct access to input fields or container elements. Moreover, `registry.category("actions")` lets you expose your component as a client action. Finally, `this.env.services.orm` provides `searchRead`, `create`, `write`, and `unlink` methods to interact with Odoo models.

## Exploring the JavaScript Code

First, let us examine the JavaScript file that defines the ICA Movie component. Furthermore, the file lives at `static/src/ica_movie/ica_movie.js` within your custom addon. Additionally, OWL and the Odoo registry import at the top set up the necessary framework pieces. Moreover, the class `IcaMovieAction` extends `Component` and implements all lifecycle and event handlers. Finally, the component registers itself with `registry.category("actions").add` to become available in the UI.

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { Component, useState, onWillStart, useRef } from "@odoo/owl";

class IcaMovieAction extends Component {
    setup() {
        this.nameRef = useRef('name');
        this.state = useState({
            partners: [],
            partner: { name: "", email: "", phone: "" },
            activeId: null,
        });
        this.resModel = "res.partner";
        this.orm = this.env.services.orm;
        onWillStart(async () => {
            await this.getAllPartners();
        });
    }

    async searchPartners(e) {
        if (e.type === "click" || e.keyCode === 13) {
            let name = this.nameRef.el.value.trim();
            await this.getAllPartners(name);
            this.nameRef.el.value = null;
        }
    }

    async getAllPartners(name) {
        const domain = [["name", "ilike", name]];
        const fields = ["id", "name", "email", "phone"];
        this.state.partners = await this.orm.searchRead(this.resModel, domain, fields);
    }

    async deletePartner(newPartner) {
        await this.orm.unlink(this.resModel, [newPartner.id]);
        this.state.partners = this.state.partners.filter((p) => p.id !== newPartner.id);
    }

    async updatePartner(newPartner) {
        this.state.partner = newPartner;
        this.state.activeId = newPartner.id;
    }

    async savePartner() {
        if (this.state.activeId) {
            await this.orm.write(this.resModel, [this.state.activeId], this.state.partner);
            this.state.activeId = false;
        } else {
            var newPartner = await this.orm.create(this.resModel, [this.state.partner]);
            this.state.partners.push({ ...this.state.partner, id: newPartner[0] });
        }
        this.state.partner = {};
    }
}

IcaMovieAction.template = "ica_movie.icaMovie";
registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);
```

First, the file starts with the Odoo module directive. Then, it imports the OWL hooks and the registry. Next, the `setup()` method creates a ref for the search input and a state object with initial values. Moreover, it defines the Odoo model (`res.partner`) and ORM API. Afterwards, it schedules `getAllPartners()` to run before the first render. Finally, the search, CRUD, and state‑update methods handle user actions.

## Delving into the XML Template

First, open `static/src/ica_movie/ica_movie.xml` to view the template. Furthermore, the file declares a `<t t-name="ica_movie.icaMovie">` template. Additionally, it uses Bootstrap classes for layout and modals. Moreover, it references `t-ref="name"` for the search input, matching the `useRef('name')` in JS. Finally, it loops over `state.partners` to render each partner row.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<templates xml:space="preserve">
  <t t-name="ica_movie.icaMovie">
    <div class="container">
      <h1 class="mt-5 mb-5">Welcome From Movies!</h1>
      <div class="d-flex justify-content-between mb-2 mt-2">
        <input t-ref="name" type="text" class="form-control"
               t-on-keyup="(e) => this.searchPartners(e)"/>
        <button class="btn btn-primary oi oi-search mx-2"
                t-on-click="searchPartners"/>
        <button type="button" class="btn btn-primary mx-2"
                data-bs-toggle="modal" data-bs-target="#exampleModal">
          New
        </button>
      </div>
      <div style="max-height: 500px; overflow-y: auto;">
        <table class="table">
          <thead class="sticky-top bg-dark text-white">
            <tr>
              <th scope="col">#</th>
              <th scope="col">Name</th>
              <th scope="col">Phone</th>
              <th scope="col">Email</th>
              <th scope="col">Actions</th>
            </tr>
          </thead>
          <tbody>
            <t t-foreach="state.partners" t-as="partner" t-key="partner.id">
              <tr>
                <th scope="row">
                  <span t-esc="partner_index + 1"/>
                </th>
                <td><span t-esc="partner.name || '-'"/></td>
                <td><span t-esc="partner.phone || '-'"/></td>
                <td><span t-esc="partner.email || '-'"/></td>
                <td>
                  <button class="btn btn-warning m-2"
                          t-on-click="() => this.updatePartner(partner)"
                          data-bs-toggle="modal"
                          data-bs-target="#exampleModal">Update
                  </button>
                  <button class="btn btn-danger m-2"
                          t-on-click="() => this.deletePartner(partner)">
                    Delete
                  </button>
                </td>
              </tr>
            </t>
          </tbody>
        </table>
      </div>
      <div class="modal fade" id="exampleModal"
           tabindex="-1" aria-labelledby="exampleModalLabel"
           aria-hidden="true">
        <div class="modal-dialog">
          <div class="modal-content">
            <div class="modal-header">
              <h5 class="modal-title" id="exampleModalLabel">New</h5>
              <button type="button" class="btn-close"
                      data-bs-dismiss="modal"
                      aria-label="Close"></button>
            </div>
            <div class="modal-body">
              <div class="container">
                <input type="text" class="form-control m-2"
                       placeholder="Name"
                       t-model="state.partner.name"/>
                <input type="text" class="form-control m-2"
                       placeholder="Email"
                       t-model="state.partner.email"/>
                <input type="text" class="form-control m-2"
                       placeholder="Phone"
                       t-model="state.partner.phone"/>
              </div>
            </div>
            <div class="modal-footer">
              <button type="button"
                      class="btn btn-secondary m-2"
                      data-bs-dismiss="modal">Close
              </button>
              <button type="button"
                      class="btn btn-primary m-2"
                      t-on-click="savePartner">Save
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </t>
</templates>
```

First, the container holds a heading and a search bar. Moreover, it binds the input to `nameRef` so JS can read and clear it. Additionally, the table displays partner data with a sticky header. Furthermore, the modal form uses `t-model` to two‑way bind form inputs to `state.partner`. Finally, the update button preloads data into the form, while save handles create or write.

## Configuring the Client Action and Menu

First, open `views/ica_movie_client_action.xml` to link the component to Odoo’s menu. Furthermore, the record of model `ir.actions.client` names the action and points to the tag `"ica_movie.movieAction"`. Additionally, the menu items under `<menuitem>` make the client action accessible in the top bar. Moreover, no parent means it appears at the root level. Finally, nested `<menuitem>` creates the “Movie” submenu.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<odoo>
  <record model="ir.actions.client" id="ica_movie_action">
    <field name="name">Movies</field>
    <field name="tag">ica_movie.movieAction</field>
  </record>
  <menuitem id="ica_movie_root" name="ICA" sequence="0"/>
  <menuitem id="movie_category" name="Movie"
            parent="ica_movie_root"
            action="ica_movie_action"
            sequence="0"/>
</odoo>
```

First, the client action record gives Odoo a handle to launch your OWL component. Moreover, the top‑level “ICA” menu groups your custom modules. Additionally, the “Movie” submenu directly opens your client action. Finally, users can click “Movie” to interact with partners in a modal‑backed grid.

## Step‑by‑Step Guide: Building Your ICA Movie OWL Component

First, we outline the key steps to create this module from scratch. Moreover, each step includes code and configuration details. Additionally, you will end with a fully working client action. Finally, this tutorial ensures you grasp both JS and XML parts.

### Step 1: Create the Addon Skeleton

First, scaffold a new addon named `ica_movie`. Furthermore, define `__manifest__.py` with dependencies on `web`. Additionally, create the folders: `static/src/ica_movie` and `views`. Moreover, ensure you add both JS and XML files to `assets.xml` under `web.assets_backend`. Finally, update `__manifest__.py` to include these assets and views.

### Step 2: Define the JS Component

First, place `ica_movie.js` in `static/src/ica_movie`. Furthermore, import OWL and registry as shown above. Additionally, define `class IcaMovieAction extends Component` with `setup()` and event methods. Moreover, register the component’s template and action tag. Finally, ensure the file ends with the registration call.

#### Step 2.1: Using onWillStart and useState

First, call `onWillStart` to fetch data before UI shows. Furthermore, assign `this.state = useState({...})` to hold partners and form data. Additionally, set `this.resModel` and `this.orm` so you can call Odoo RPC.

#### Step 2.2: Handling User Events with useRef

First, declare `this.nameRef = useRef('name')` to attach to the search input. Moreover, in `searchPartners(e)`, read `this.nameRef.el.value`. Additionally, trim and pass the search term to `getAllPartners()`. Finally, clear the input after search for better UX.

### Step 3: Build the XML Template

First, create `ica_movie.xml` under `static/src/ica_movie`. Furthermore, define the `<t t-name>` with container, input, table, and modal. Additionally, use `t-on-click`, `t-on-keyup`, and `t-model` directives. Moreover, keep your markup simple and leverage Bootstrap for styling. Finally, verify your IDs and refs match exactly those in JS.

#### Step 3.1: Adding Bootstrap Modal and Table

First, use Bootstrap classes like `.modal`, `.table`, and `.sticky-top`. Furthermore, ensure the modal markup follows Bootstrap 5 structure. Additionally, place `data-bs-toggle="modal"` and `data-bs-target="#exampleModal"` on the New button. Finally, wrap form fields in the modal body with `t-model` for two‑way binding.

### Step 4: Configure the Action and Menu

First, define `ica_movie_client_action.xml` in `views`. Furthermore, create an `ir.actions.client` record with your tag. Additionally, add top‑level and child menu items. Moreover, set `sequence` to control ordering. Finally, update your manifest to load this view file.

## Advanced Tips and Best Practices

First, always keep your code DRY and modular. Furthermore, split large components into smaller ones when your UI grows. Additionally, add error handling around RPC calls to improve resilience. Moreover, use CSS classes sparingly and rely on Bootstrap or your theme. Finally, write clear comments so other developers can follow your logic.

### Improving Code Readability

First, choose meaningful names for refs and state keys. Furthermore, keep methods short and purposeful. Additionally, group related functions together. Moreover, align your XML indentation consistently. Finally, document each hook and API call.

### Optimizing Keyphrase Distribution

First, sprinkle “Odoo Owl tutorial” and synonyms like “Owl component guide” in headings and text. Furthermore, avoid overuse to maintain readability. Additionally, ensure at least one transition word per sentence to guide readers. Moreover, preview your post in an SEO tool to check density. Finally, balance keyphrase usage across sections.

## Conclusion and Next Steps

First, you have built a custom ICA Movie client action using an Odoo Owl tutorial approach. Furthermore, you now understand how to use OWL hooks, state, and registry. Additionally, you saw how XML templates and client actions tie everything together. Moreover, you can extend this pattern to other models and views. Finally, explore advanced OWL features like scoped slots and context to deepen your skills.

### Further Reading and External Links

First, for more on OWL, check the [Odoo OWL docs](https://www.odoo.com/documentation/16.0/developer/reference/frontend/owl.html). Furthermore, explore the official Odoo GitHub to see community addons. Additionally, join the Odoo forums to ask questions and share your work. Finally, keep practicing to master Odoo OWL development!