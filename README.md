# Odoo 17 OWL tutorial: Create a Custom Movie Client Action

Odoo 17 OWL tutorial guides you step by step to build a custom movie client action in Odoo 17. Moreover, this OWL component guide shows you how to set up controllers, models, JavaScript components, XML templates, and service patches. Furthermore, you will learn to use hooks like `onWillStart`, `useState`, `useRef`, and more. In addition, you can visit the [Odoo OWL docs](https://www.odoo.com/documentation/17.0/developer/reference/frontend/owl.html) for deeper reference.

## Prerequisites for Odoo OWL development

First, ensure you run Odoo 17 with developer mode enabled. Next, you need basic Python and JavaScript knowledge. Then, install Node.js and Yarn to manage OWL assets. Moreover, set up your custom add‑on directory under `addons/ica_movie`. Finally, confirm you can restart Odoo and see logs in real time.

### File structure overview

First, create this layout under `custom-vf/ica_movie`  
```
ica_movie/
├── __manifest__.py
├── controllers/
│   └── main.py
├── models/
│   └── res_partner.py
├── static/
│   └── src/
│       └── ica_movie/
│           ├── ica_movie.js
│           ├── ica_movie.xml
│           └── IcaMovieActionER.js
└── views/
    └── ica_movie_client_action.xml
```  
Next, we explain each file.

## Odoo controller setup in Python

First, we handle RPC routes and bus notifications. Then, define your HTTP controller in `controllers/main.py`. Moreover, you use `@http.route` to expose JSON endpoints.

```python
# controllers/main.py
from odoo import http
from odoo.http import request

class MainController(http.Controller):
    @http.route('/rpc/login', type='json', auth='user')
    def rpc_login(self, username, password, **kw):
        print("RPC login called:", username, password, kw)
        return {"message": "successfully"}

    @http.route('/ica/send-bus', type='json', auth='user')
    def send_bus(self, **kw):
        print("Sending bus payload:", kw)
        request.env['bus.bus']._sendone(
            'ica-movie-channel',
            'ica-movie-channel/sending-message',
            kw
        )
        return True
```

#### Explanation of controller code

- First, we import `http` and `request`.  
- Next, the `rpc_login` method logs in via RPC.  
- Then, `send_bus` sends real‑time messages via the bus service on channel `ica-movie-channel`.  
- Finally, both methods return JSON to the OWL component.

## Extending the res.partner model

First, add a custom Python method on `res.partner`. Then, call it via `ORM.call()` from JavaScript.

```python
# models/res_partner.py
from odoo import api, fields, models

class Respartner(models.Model):
    _inherit = 'res.partner'

    def action_class_from_json(self, name, email):
        print("Called from OWL:", self, name, email)
```

#### Explanation of model patch

- First, we inherit `res.partner`.  
- Next, we define `action_class_from_json`.  
- Then, OWL can call this method via RPC.  
- Finally, we log parameters to confirm the call.

## JavaScript OWL component code

Next, we build the main OWL component in `static/src/ica_movie/ica_movie.js`. Moreover, we import OWL hooks, services, and registry.

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { Component, useState, onWillStart, useRef } from "@odoo/owl";
import { ConfirmationDialog } from "@web/core/confirmation_dialog/confirmation_dialog";
import { Layout } from "@web/search/layout";
import { Notebook } from "@web/core/notebook/notebook";
import { useAutofocus } from "@web/core/utils/hooks";
import { cookie } from "@web/core/browser/cookie";
import { routeToUrl } from "@web/core/browser/router_service";
import { browser } from "@web/core/browser/browser";

export default class IcaMovieAction extends Component {
  static template = "ica_movie.icaMovie";
  static components = { Layout, Notebook };

  setup() {
    // Input refs
    this.nameRef = useAutofocus({ refName: "name" });
    this.inputRef = useRef("input-box");

    // Reactive state
    this.state = useState({
      partners: [],
      partner: { name: "", email: "", phone: "" },
      activeId: null,
      todos: [],
      darkTheme: false,
      image: null,
      messages: [],
    });

    // Odoo services
    this.orm = this.env.services.orm;
    this.rpcService = this.env.services.rpc;
    this.httpService = this.env.services.http;
    this.dialog = this.env.services.dialog;
    this.effectService = this.env.services.effect;
    this.userService = this.env.services.user;
    this.busService = this.env.services.bus_service;
    this.titleService = this.env.services.title;
    this.routerService = this.env.services.router;
    this.companyService = this.env.services.company;

    // Bus channel
    this.busService.addChannel("ica-movie-channel");
    this.busService.subscribe(
      "ica-movie-channel/sending-message",
      (payload) => {
        this.state.messages.push(payload.message);
      }
    );

    // Pre‑render setup
    onWillStart(async () => {
      await this.getAllPartners();
      this.initTheme();
      this.changeTitle();
      this.loadUserAvatar();
    });
  }

  // Toggle dark/light theme
  switchTheme() {
    const current = cookie.get("darkTheme") === "true";
    cookie.set("darkTheme", !current);
    this.state.darkTheme = !current;
  }

  // Initialize theme from cookie
  initTheme() {
    this.state.darkTheme = cookie.get("darkTheme") === "true";
  }

  // Change page title
  changeTitle() {
    this.titleService.setParts({ zopenerp: "Movie Action" });
  }

  // Load current user avatar
  loadUserAvatar() {
    const id = this.userService.partnerId;
    this.state.image = `/web/image?model=res.partner&id=${id}&field=avatar_128`;
  }

  // Search or click handler
  async searchPartners(e) {
    if (e.type === "click" || e.keyCode === 13) {
      const name = this.nameRef.el.value.trim();
      await this.getAllPartners(name);
      this.nameRef.el.value = "";
    }
  }

  // Fetch partners via ORM
  async getAllPartners(name = "") {
    const domain = [["name", "ilike", name]];
    const fields = ["id", "name", "email", "phone"];
    this.state.partners = await this.orm.searchRead(
      "res.partner",
      domain,
      fields,
      { order: "id desc" }
    );
  }

  // Delete with confirmation and effect
  deletePartner(partner) {
    this.dialog.add(
      ConfirmationDialog,
      {
        title: "Delete Partner",
        body: `Delete ${partner.name}?`,
        confirm: async () => {
          await this.orm.unlink("res.partner", [partner.id]);
          this.state.partners = this.state.partners.filter(
            (p) => p.id !== partner.id
          );
          this.effectService.add({
            type: "rainbow_man",
            message: "Record deleted successfully.",
          });
        },
        cancel: () => {},
      },
      {
        onClose: () => console.log("Dialog closed"),
      }
    );
  }

  // Prepare update form
  async updatePartner(partner) {
    this.state.partner = partner;
    this.state.activeId = partner.id;
  }

  // Save or update partner
  async savePartner() {
    if (this.state.activeId) {
      await this.orm.write("res.partner", [this.state.activeId], this.state.partner);
      this.state.activeId = null;
      this.env.services.notification.add("Partner updated", {
        title: "Success",
        type: "success",
      });
    } else {
      const newIds = await this.orm.create("res.partner", [this.state.partner]);
      this.state.partners.push({
        ...this.state.partner,
        id: newIds[0],
      });
      this.env.services.notification.add("Partner created", {
        title: "Success",
        type: "success",
      });
    }
    this.state.partner = { name: "", email: "", phone: "" };
  }

  // Call custom ORM method
  async callOrmMethod(partner) {
    await this.orm.call(
      "res.partner",
      "action_class_from_json",
      [[partner.id]],
      { name: partner.name, email: partner.email }
    );
  }

  // Send real‑time message
  async sendMessage() {
    const msg = this.inputRef.el.value.trim();
    if (msg) {
      await this.rpcService("/ica/send-bus", { message: msg });
      this.inputRef.el.value = "";
    }
  }

  // Fetch external todo list
  async getTodoList() {
    const todos = await this.httpService.get(
      "https://jsonplaceholder.typicode.com/todos"
    );
    this.state.todos = todos;
  }

  // Change router query params
  changeRouter() {
    const cur = this.routerService.current.search;
    cur.debug = !cur.debug;
    cur.darkTheme = !cur.darkTheme;
    browser.location.href = browser.location.origin + routeToUrl(this.routerService.current);
  }

  // Toggle control panel display
  getCompany() {
    this.display.controlPanel.topRight = false;
  }
}

registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);
```

#### Explanation of JavaScript code

- First, we import OWL hooks and Odoo services.  
- Next, we define `IcaMovieAction` with `setup()`.  
- Then, we use `onWillStart` to fetch data and set theme before render.  
- Moreover, we handle search, CRUD, RPC, HTTP, notifications, router, and company services.  
- Finally, we register the action under tag `ica_movie.movieAction`.

## XML template for the OWL component

First, we define the UI in `static/src/ica_movie/ica_movie.xml`. Then, we use `<Layout>` and `<Notebook>` components from Odoo.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<templates xml:space="preserve">
  <t t-name="ica_movie.icaMovie">
    <div t-attf-class="{{ state.darkTheme ? 'bg-dark text-white' : '' }}">
      <Layout display="display">
        <!-- Header with theme toggle and avatar -->
        <div class="container d-flex justify-content-between mt-5">
          <button class="btn btn-primary" t-on-click="switchTheme">
            Switch Theme
          </button>
          <img t-att-src="state.image" class="rounded-circle" />
        </div>

        <!-- Notebook pages -->
        <div class="container mt-5">
          <Notebook orientation="'horizontal'">
            <!-- Messages tab -->
            <t t-set-slot="page_0" title="'Messages'" isVisible="true">
              <h2>Real‑time Messages</h2>
              <div style="max-height:200px;overflow-y:auto;">
                <t t-foreach="state.messages" t-as="msg" t-key="msg_index">
                  <div class="alert alert-info m-2"><t t-esc="msg"/></div>
                </t>
              </div>
              <div class="d-flex">
                <input t-ref="input-box" class="form-control me-2" placeholder="Type a message..."/>
                <button class="btn btn-success" t-on-click="sendMessage">Send</button>
              </div>
            </t>

            <!-- Movies tab -->
            <t t-set-slot="page_1" title="'Movies'" isVisible="true">
              <h2>Movie Partners</h2>
              <div class="d-flex mb-3">
                <input t-ref="name" type="text" class="form-control me-2"
                       placeholder="Search by name..." t-on-keyup="searchPartners"/>
                <button class="btn btn-outline-primary me-2" t-on-click="searchPartners">
                  Search
                </button>
                <button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#exampleModal">
                  New Partner
                </button>
              </div>
              <div style="max-height:400px;overflow-y:auto;">
                <table class="table table-striped">
                  <thead class="table-dark">
                    <tr>
                      <th>#</th>
                      <th>Name</th>
                      <th>Phone</th>
                      <th>Email</th>
                      <th>Actions</th>
                    </tr>
                  </thead>
                  <tbody>
                    <t t-foreach="state.partners" t-as="partner" t-key="partner.id">
                      <tr>
                        <td><t t-esc="partner_index+1"/></td>
                        <td><t t-esc="partner.name || '-'"/></td>
                        <td><t t-esc="partner.phone || '-'"/></td>
                        <td><t t-esc="partner.email || '-'"/></td>
                        <td>
                          <button class="btn btn-sm btn-warning me-1"
                                  t-on-click="() => updatePartner(partner)"
                                  data-bs-toggle="modal" data-bs-target="#exampleModal">
                            Edit
                          </button>
                          <button class="btn btn-sm btn-danger me-1"
                                  t-on-click="() => deletePartner(partner)">
                            Delete
                          </button>
                          <button class="btn btn-sm btn-info"
                                  t-on-click="() => callOrmMethod(partner)">
                            Call ORM
                          </button>
                        </td>
                      </tr>
                    </t>
                  </tbody>
                </table>
              </div>

              <!-- Partner modal -->
              <div class="modal fade" id="exampleModal" tabindex="-1" aria-hidden="true">
                <div class="modal-dialog">
                  <div class="modal-content">
                    <div class="modal-header">
                      <h5 class="modal-title">Partner Form</h5>
                      <button type="button" class="btn-close" data-bs-dismiss="modal"/>
                    </div>
                    <div class="modal-body">
                      <input type="text" class="form-control mb-2" placeholder="Name"
                             t-model="state.partner.name"/>
                      <input type="text" class="form-control mb-2" placeholder="Email"
                             t-model="state.partner.email"/>
                      <input type="text" class="form-control mb-2" placeholder="Phone"
                             t-model="state.partner.phone"/>
                    </div>
                    <div class="modal-footer">
                      <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">
                        Close
                      </button>
                      <button type="button" class="btn btn-primary"
                              data-bs-dismiss="modal" t-on-click="savePartner">
                        Save
                      </button>
                    </div>
                  </div>
                </div>
              </div>
            </t>

            <!-- RPC Service tab -->
            <t t-set-slot="page_2" title="'RPC Service'" isVisible="true">
              <h2>Test RPC Login</h2>
              <button class="btn btn-outline-secondary" t-on-click="callingRPCService">
                RPC Login
              </button>
            </t>

            <!-- HTTP Service tab -->
            <t t-set-slot="page_3" title="'HTTP Service'" isVisible="true">
              <h2>Load Todo List</h2>
              <button class="btn btn-outline-secondary mb-3" t-on-click="getTodoList">
                Fetch Todos
              </button>
              <div style="max-height:300px;overflow-y:auto;">
                <table class="table">
                  <thead class="table-dark">
                    <tr><th>#</th><th>Title</th><th>Done</th></tr>
                  </thead>
                  <tbody>
                    <t t-foreach="state.todos" t-as="todo" t-key="todo.id">
                      <tr>
                        <td><t t-esc="todo_index+1"/></td>
                        <td><t t-esc="todo.title"/></td>
                        <td><t t-esc="todo.completed"/></td>
                      </tr>
                    </t>
                  </tbody>
                </table>
              </div>
            </t>

            <!-- Other Services tab -->
            <t t-set-slot="page_4" title="'Other Services'" isVisible="true">
              <h2>Router & Company</h2>
              <button class="btn btn-outline-primary me-2" t-on-click="changeRouter">
                Toggle Router
              </button>
              <button class="btn btn-outline-primary" t-on-click="getCompany">
                Hide CP
              </button>
            </t>
          </Notebook>
        </div>
      </Layout>
    </div>
  </t>
</templates>
```

#### Explanation of XML template

- First, we wrap content in `<Layout>` for Odoo top bar.  
- Next, we use `<Notebook>` for tabs/pages.  
- Then, each `<t t-set-slot>` defines a page with title and content.  
- Moreover, we bind input and buttons via `t-ref`, `t-on-click`, and `t-model`.  
- Finally, we link the modal form to save and update partners.

##  Patching lifecycle hooks with ER file

First, extend the component to log OWL lifecycle events. Then, place this in `IcaMovieActionER.js`.

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import IcaMovieAction from "./ica_movie";
import { patch } from "@web/core/utils/patch";
import {
  onWillStart,
  onWillRender,
  onRendered,
  onMounted,
  onWillDestroy,
} from "@odoo/owl";

patch(IcaMovieAction.prototype, {
  setup() {
    super.setup(...arguments);
    onWillStart(() => console.log("onWillStart"));
    onWillRender(() => console.log("onWillRender"));
    onRendered(() => console.log("onRendered"));
    onMounted(() => console.log("onMounted"));
    onWillDestroy(() => console.log("onWillDestroy"));
  },

  async callOrmMethod(partner) {
    await this.orm.call(
      "res.partner",
      "action_class_from_json",
      [[partner.id]],
      { name: partner.name, email: partner.email }
    );
  },

  async callingRPCService() {
    await this.rpcService("/rpc/login", {
      username: "admin",
      password: "admin",
    });
  },

  async searchPartners(...args) {
    const result = await super.searchPartners(...args);
    console.log("Extended searchPartners");
    return result;
  },
});

registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);
```

#### Explanation of patch file

- First, we import OWL lifecycle hooks.  
- Next, we patch `setup()` to log each hook.  
- Then, we override `callOrmMethod`, `callingRPCService`, and `searchPartners` to add behavior.  
- Finally, we re‑register the action.

##  Client action and menu XML

First, define the client action and menu in `views/ica_movie_client_action.xml`.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<odoo>
  <record model="ir.actions.client" id="ica_movie_action">
    <field name="name">Movies</field>
    <field name="tag">ica_movie.movieAction</field>
  </record>
  <menuitem id="ica_movie_root" name="ICA" sequence="10"/>
  <menuitem id="movie_category" name="Movie"
            parent="ica_movie_root"
            action="ica_movie_action"
            sequence="5"/>
</odoo>
```

#### Explanation of client action XML

- First, `ir.actions.client` links to your OWL tag.  
- Next, top‑level menu “ICA” groups your addons.  
- Then, “Movie” submenu launches the action.  
- Finally, users see “ICA > Movie” in the top bar.

##  Addon manifest configuration

First, list your files and assets in `__manifest__.py`.

```python
{
    "name": "ICA Movie",
    "version": "1.0",
    "category": "Custom",
    "summary": "Custom Movie Partner OWL Action",
    "depends": ["base", "web", "mail"],
    "license": "LGPL-3",
    "data": [
        "views/ica_movie_client_action.xml",
    ],
    "assets": {
        "web.assets_backend": [
            "/ica_movie/static/src/ica_movie/ica_movie.js",
            "/ica_movie/static/src/ica_movie/ica_movie.xml",
            "/ica_movie/static/src/ica_movie/IcaMovieActionER.js",
        ]
    },
}
```

#### Explanation of manifest

- First, we list addon info and dependencies.  
- Next, we include the client action view.  
- Then, we register JavaScript and XML under `web.assets_backend`.  
- Finally, Odoo loads assets when rendering the back end.

##  Testing and deploying your OWL tutorial guide

First, restart your Odoo server with `-u ica_movie`. Then, open the top bar and click **ICA > Movie**. Next, you should see the dark‑theme toggle, notebook tabs, and partner list. Moreover, test each feature:

- Toggle theme and reload.  
- Send a real‑time message and watch it appear in **Messages**.  
- Search, create, update, and delete partners.  
- Click **Call ORM** to trigger the Python method.  
- Click **RPC Login** to test your login route.  
- Fetch todos via HTTP service.  
- Toggle router debug flags and refresh.  
- Hide the control panel via **Other Services**.

## Advanced tips and next steps

First, you can split the component into child components for each tab. Moreover, you can add error handling around RPC and HTTP calls. Additionally, you can theme the modal or table via SCSS. Furthermore, you can integrate real‑time chat using `mail.thread`. Finally, you can release this addon on GitHub to share your OWL tutorial hack.

## Conclusion

In this Odoo 17 OWL tutorial, you learned how to build and configure a custom movie client action from scratch. Moreover, you saw how to set up controllers, models, OWL components, XML templates, and patches. Additionally, you learned to use OWL hooks, Odoo services, and real‑time bus notifications. Furthermore, you followed best practices for code structure, state management, and UI layout. Finally, you can now apply this OWL guide to your own projects and extend Odoo with powerful, reactive components.