# Odoo 17 OWL custom client action: step‑by‑step tutorial

First, build custom client action in Odoo 17 OWL by updating your add‑on manifest, adding HTTP controllers, and extending the `res.partner` model. Next, we create a reactive OWL component with tabs, real‑time bus messages, RPC routes, and CRUD operations. Then, we build standalone pages with Bootstrap, FontAwesome, and a Todo app. Moreover, we register menus and actions for both back‑end and public views. Finally, we link to the [Odoo OWL docs](https://www.odoo.com/documentation/17.0/developer/reference/frontend/owl.html) to deepen your understanding.

## create custom client action – update module manifest

First, build custom client action requires declaring assets and data files in `__manifest__.py`. Next, we add a bundle for the standalone OWL app.

```diff
diff --git a/ica_movie/__manifest__.py b/ica_movie/__manifest__.py
index 96ac2c9..6ad96a0 100644
--- a/ica_movie/__manifest__.py
+++ b/ica_movie/__manifest__.py
@@ -1,13 +1,29 @@
 {
-   "name": "ICA  Movie",
+   "name": "ICA Movie",
    "depends": ["base", "web", "mail"],
    "license": "LGPL-3",
    "data": [
-   "views/ica_movie_client_action.xml",
+   "views/ica_movie_client_action.xml",
+   "views/template.xml",
    ],
    "assets": {
      "web.assets_backend": [
-       "/ica_movie/static/src/**/*",
+       "/ica_movie/static/src/ica_movie/*",
      ],
+     "ica_movie.assets_standalone_app": [
+         ("include", "web._assets_helpers"),
+         "web/static/src/scss/pre_variables.scss",
+         "web/static/lib/bootstrap/scss/_variables.scss",
+         ("include", "web._assets_bootstrap"),
+         ("include", "web._assets_core"),
+         "web/static/src/libs/fontawesome/css/font-awesome.css",
+         "web/static/lib/odoo_ui_icons/*",
+         "ica_movie/static/src/standalone_app/**/*.js",
+         "ica_movie/static/src/standalone_app/**/*.xml",
+         "ica_movie/static/src/standalone_app/**/*.scss",
+     ],
    }
 }
```

#### Explanation of manifest changes

- First, we include `"views/template.xml"` to render the public standalone page.  
- Next, we scope back‑end OWL assets under `web.assets_backend`.  
- Then, we define `ica_movie.assets_standalone_app` to bundle Bootstrap, FontAwesome, Odoo helpers, and our standalone‑app files.  
- Finally, Odoo will compile these assets for both the back‑end and the public page.

## extend HTTP controllers for custom client action

First, build custom client action needs server routes for RPC and public pages. Next, we update `controllers/main.py`:

```python
# controllers/main.py
from odoo import http
from odoo.http import request
import odoo

class MainController(http.Controller):
    @http.route('/rpc/login', type='json', auth='user')
    def rpc_login(self, username, password, **kw):
        print(kw)
        print(username, password)
        return {"message": "successfully"}

    @http.route('/ica/send-bus', type='json', auth='user')
    def send_bus(self, **kw):
        request.env['bus.bus']._sendone(
            'ica-movie-channel',
            'ica-movie-channel/sending-message',
            kw
        )
        return True

    @http.route("/ica-movie/standalone_app", auth="public")
    def standalone_app(self, **kw):
        get_frontend_session_info: dict = request.env['ir.http'].session_info()
        return request.render(
            'ica_movie.standalone_app',
            get_frontend_session_info
        )
```

#### Explanation of controller code

- First, we expose `/rpc/login` to test RPC calls.  
- Next, we broadcast real‑time messages on channel `ica-movie-channel` in `send_bus`.  
- Then, we define a public route `/ica-movie/standalone_app` to serve our standalone HTML shell.  
- Finally, we pass CSRF and session info to the QWeb template.

## extend res.partner model for client action

First, build custom client action needs a server‑side method. Next, we patch the model in `models/res_partner.py`:

```python
# models/res_partner.py
from odoo import api, fields, models

class Respartner(models.Model):
    _inherit = 'res.partner'

    def action_class_from_json(self, name, email):
        print("*" * 80)
        print("Partner:", self)
        print("Name:", name, "Email:", email)
```

#### Explanation of model extension

- First, we inherit `res.partner`.  
- Next, we define `action_class_from_json` to accept JSON parameters.  
- Then, OWL can call this method via `orm.call()`.  
- Finally, we log inputs for debugging.

## build OWL client action component

First, build custom client action requires an OWL component under `static/src/ica_movie/ica_movie.js`. Next, we import hooks and services:

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
    this.nameRef = useAutofocus({ refName: "name" });
    this.inputRef = useRef("input-box");
    this.display = { controlPanel: { topRight: true } };
    this.state = useState({
      partners: [],
      partner: { name: "", email: "", phone: "" },
      activeId: null,
      todos: [],
      darkTheme: false,
      image: null,
      messages: [],
    });
    this.resModel = "res.partner";
    this.orm = this.env.services.orm;
    this.rpcService = this.env.services.rpc;
    this.httpService = this.env.services.http;
    this.dialog = this.env.services.dialog;
    this.effect = this.env.services.effect;
    this.user = this.env.services.user;
    this.bus = this.env.services.bus_service;
    this.title = this.env.services.title;
    this.router = this.env.services.router;
    this.company = this.env.services.company;

    this.bus.addChannel("ica-movie-channel");
    this.bus.subscribe("ica-movie-channel/sending-message", (payload) => {
      this.state.messages.push(payload.message);
    });

    onWillStart(async () => {
      await this.loadPartners();
      this.initTheme();
      this.setTitle();
      this.loadAvatar();
    });
  }

  initTheme() {
    this.state.darkTheme = cookie.get("darkTheme") === "true";
  }

  switchTheme() {
    const next = !this.state.darkTheme;
    cookie.set("darkTheme", next);
    this.state.darkTheme = next;
  }

  setTitle() {
    this.title.setParts({ zopenerp: "Movies" });
  }

  loadAvatar() {
    const uid = this.user.partnerId;
    this.state.image = `/web/image?model=res.partner&id=${uid}&field=avatar_128`;
  }

  async loadPartners(name = "") {
    this.state.partners = await this.orm.searchRead(
      this.resModel,
      [["name", "ilike", name]],
      ["id", "name", "email", "phone"],
      { order: "id desc" }
    );
  }

  async searchPartners(e) {
    if (e.type === "click" || e.keyCode === 13) {
      const name = this.nameRef.el.value.trim();
      await this.loadPartners(name);
      this.nameRef.el.value = "";
    }
  }

  async savePartner() {
    if (this.state.activeId) {
      await this.orm.write(
        this.resModel,
        [this.state.activeId],
        this.state.partner
      );
      this.state.activeId = null;
      this.env.services.notification.add("Partner updated", {
        title: "Success",
        type: "success",
      });
    } else {
      const [newId] = await this.orm.create(this.resModel, [this.state.partner]);
      this.state.partners.push({ ...this.state.partner, id: newId });
      this.env.services.notification.add("Partner created", {
        title: "Success",
        type: "success",
      });
    }
    this.state.partner = { name: "", email: "", phone: "" };
  }

  deletePartner(partner) {
    this.dialog.add(
      ConfirmationDialog,
      {
        title: "Delete Partner",
        body: `Delete ${partner.name}?`,
        confirm: async () => {
          await this.orm.unlink(this.resModel, [partner.id]);
          this.state.partners = this.state.partners.filter(
            (p) => p.id !== partner.id
          );
          this.effect.add({
            type: "rainbow_man",
            message: "Record deleted successfully.",
          });
        },
      },
      { onClose: () => console.log("Dialog closed") }
    );
  }

  async callOrmMethod(partner) {
    await this.orm.call(this.resModel, "action_class_from_json", [[partner.id]], {
      name: partner.name,
      email: partner.email,
    });
  }

  async sendMessage() {
    const msg = this.inputRef.el.value.trim();
    if (msg) {
      await this.rpcService("/ica/send-bus", { message: msg });
      this.inputRef.el.value = "";
    }
  }

  async fetchTodos() {
    this.state.todos = await this.httpService.get(
      "https://jsonplaceholder.typicode.com/todos"
    );
  }

  toggleRouter() {
    const cur = this.router.current.search;
    cur.debug = !cur.debug;
    cur.darkTheme = !cur.darkTheme;
    browser.location.href =
      browser.location.origin + routeToUrl(this.router.current);
  }

  hideControlPanel() {
    this.display.controlPanel.topRight = false;
  }
}

registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);
```

#### setup and services

- First, we import OWL hooks (`useState`, `onWillStart`, `useRef`, `useAutofocus`).  
- Next, we inject Odoo services: `orm`, `rpc`, `http`, `bus_service`, `dialog`, `effect`, `user`, `title`, `router`, and `company`.  
- Then, we subscribe to real‑time bus messages on channel `ica-movie-channel`.  
- Finally, we pre‑load partners, theme, title, and avatar before rendering.

#### CRUD and interactivity

- First, `loadPartners` fetches partners via the ORM.  
- Next, `searchPartners` triggers on click or Enter key.  
- Then, `savePartner` handles both create and update cases.  
- Moreover, `deletePartner` shows a confirmation dialog before deletion.  
- Additionally, `callOrmMethod` invokes our custom Python method.  
- Finally, `sendMessage` and `fetchTodos` test RPC and HTTP services.

## define QWeb template for client action

First, build custom client action uses a QWeb template in `static/src/ica_movie/ica_movie.xml`:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<templates xml:space="preserve">
  <t t-name="ica_movie.icaMovie">
    <div t-att-class="{{ state.darkTheme ? 'bg-dark text-white' : '' }}">
      <Layout display="display">
        <div class="container d-flex justify-content-between mt-4">
          <button class="btn btn-primary" t-on-click="switchTheme">
            Switch Theme
          </button>
          <img t-att-src="state.image" class="rounded-circle" width="40"/>
        </div>
        <div class="container mt-4">
          <Notebook orientation="'horizontal'">
            <!-- Messages Tab -->
            <t t-set-slot="page_0" title="'Messages'" isVisible="true">
              <h3>Real‑time Messages</h3>
              <div style="max-height:200px;overflow-y:auto;">
                <t t-foreach="state.messages" t-as="m" t-key="m_index">
                  <div class="alert alert-info m-2"><t t-esc="m"/></div>
                </t>
              </div>
              <div class="d-flex">
                <input t-ref="input-box" class="form-control me-2" placeholder="Type a message..."/>
                <button class="btn btn-success" t-on-click="sendMessage">Send</button>
              </div>
            </t>

            <!-- Movies Tab -->
            <t t-set-slot="page_1" title="'Movies'" isVisible="true">
              <h3>Movie Partners</h3>
              <div class="d-flex mb-3">
                <input
                  t-ref="name"
                  type="text"
                  class="form-control me-2"
                  placeholder="Search by name..."
                  t-on-keyup="searchPartners"
                />
                <button class="btn btn-outline-primary me-2" t-on-click="searchPartners">Search</button>
                <button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#partnerModal">New Partner</button>
              </div>
              <div style="max-height:400px;overflow-y:auto;">
                <table class="table table-striped">
                  <thead class="table-dark">
                    <tr><th>#</th><th>Name</th><th>Phone</th><th>Email</th><th>Actions</th></tr>
                  </thead>
                  <tbody>
                    <t t-foreach="state.partners" t-as="p" t-key="p.id">
                      <tr>
                        <td><t t-esc="p_index+1"/></td>
                        <td><t t-esc="p.name||'-'"/></td>
                        <td><t t-esc="p.phone||'-'"/></td>
                        <td><t t-esc="p.email||'-'"/></td>
                        <td>
                          <button class="btn btn-sm btn-warning me-1" t-on-click="() => updatePartner(p)" data-bs-toggle="modal" data-bs-target="#partnerModal">Edit</button>
                          <button class="btn btn-sm btn-danger me-1" t-on-click="() => deletePartner(p)">Delete</button>
                          <button class="btn btn-sm btn-info" t-on-click="() => callOrmMethod(p)">Call ORM</button>
                        </td>
                      </tr>
                    </t>
                  </tbody>
                </table>
              </div>

              <!-- Modal for create/update -->
              <div class="modal fade" id="partnerModal" tabindex="-1" aria-hidden="true">
                <div class="modal-dialog">
                  <div class="modal-content">
                    <div class="modal-header">
                      <h5 class="modal-title">Partner Form</h5>
                      <button type="button" class="btn-close" data-bs-dismiss="modal"/>
                    </div>
                    <div class="modal-body">
                      <input type="text" class="form-control mb-2" placeholder="Name" t-model="state.partner.name"/>
                      <input type="text" class="form-control mb-2" placeholder="Email" t-model="state.partner.email"/>
                      <input type="text" class="form-control mb-2" placeholder="Phone" t-model="state.partner.phone"/>
                    </div>
                    <div class="modal-footer">
                      <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
                      <button type="button" class="btn btn-primary" data-bs-dismiss="modal" t-on-click="savePartner">Save</button>
                    </div>
                  </div>
                </div>
              </div>
            </t>

            <!-- RPC Service Tab -->
            <t t-set-slot="page_2" title="'RPC Service'" isVisible="true">
              <h3>Test RPC Login</h3>
              <button class="btn btn-outline-secondary" t-on-click="callingRPCService">RPC Login</button>
            </t>

            <!-- HTTP Service Tab -->
            <t t-set-slot="page_3" title="'HTTP Service'" isVisible="true">
              <h3>Load Todo List</h3>
              <button class="btn btn-outline-secondary mb-3" t-on-click="fetchTodos">Fetch Todos</button>
              <div style="max-height:300px;overflow-y:auto;">
                <table class="table">
                  <thead class="table-dark"><tr><th>#</th><th>Title</th><th>Done</th></tr></thead>
                  <tbody>
                    <t t-foreach="state.todos" t-as="todo" t-key="todo.id">
                      <tr><td><t t-esc="todo_index+1"/></td><td><t t-esc="todo.title"/></td><td><t t-esc="todo.completed"/></td></tr>
                    </t>
                  </tbody>
                </table>
              </div>
            </t>

            <!-- Other Services Tab -->
            <t t-set-slot="page_4" title="'Other Services'" isVisible="true">
              <h3>Router & Company</h3>
              <button class="btn btn-outline-primary me-2" t-on-click="toggleRouter">Toggle Router</button>
              <button class="btn btn-outline-primary" t-on-click="hideControlPanel">Hide CP</button>
            </t>
          </Notebook>
        </div>
      </Layout>
    </div>
  </t>
</templates>
```

#### Explanation of template

- First, we wrap content in `<Layout>` for Odoo’s top bar.  
- Next, we use `<Notebook>` to create tabs for messages, partners, RPC, HTTP, and services.  
- Then, we bind inputs via `t-ref`, `t-on-click`, and `t-model`.  
- Moreover, we include a Bootstrap modal for creating and updating partners.  
- Finally, we adapt styles based on `state.darkTheme`.

## patch OWL lifecycle – extend IcaMovieAction

First, build custom client action often requires logging lifecycle events. Next, we patch in `static/src/ica_movie/IcaMovieActionER.js`:

```javascript
/** @odoo-module **/
import { registry } from "@web/core/registry";
import IcaMovieAction from "./ica_movie";
import { patch } from "@web/core/utils/patch";
import {
  onWillRender,
  onWillStart,
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
    onWillDestroy(() => console.log("cleanup on destroy"));
    await this.orm.call(
      this.resModel,
      "action_class_from_json",
      [[partner.id]],
      { name: partner.name, email: partner.email }
    );
  },

  async callingRPCService() {
    console.log("RPC login click");
    await this.rpcService("/rpc/login", {
      username: "username",
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

#### Explanation of patch

- First, we import OWL lifecycle hooks.  
- Next, we call `super.setup()` and attach loggers to each hook.  
- Then, we override methods to add custom behavior.  
- Finally, we re‑register the action.

## build dynamic list and grid view components

First, build custom client action often needs reusable view modes. Next, we define a row renderer and wrapper components.

### CustomerList row component

```javascript
// static/src/customer_list/customer_list.js
/** @odoo-module **/
import { Component } from "@odoo/owl";

export class CustomerList extends Component {
  static template = "ica_movie.CustomerList";
  static props = {};
}
```

```xml
<!-- static/src/customer_list/customer_list.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.CustomerList">
    <t t-set="partner" t-value="props.partner"/>
    <tr>
      <th scope="row"><t t-esc="props.index"/></th>
      <td>
        <t t-if="partner.imageURL">
          <img t-att-src="partner.imageURL"/>
        </t>
        <t t-else=""><span>-</span></t>
      </td>
      <td><t t-esc="partner.name||'-'"/></td>
      <td><t t-esc="partner.email||'-'"/></td>
    </tr>
  </t>
</templates>
```

### ListViewComponent wrapper

```javascript
// static/src/list_view/list_view.js
/** @odoo-module **/
import { Component } from "@odoo/owl";
import { ScrollableComponent } from "../standalone_app/components/scrollable_component/scrollable_component";
import { CustomerList } from "../customer_list/customer_list";

export class ListViewComponent extends Component {
  static template = "ica_movie.ListViewComponent";
  static props = {};
  static components = { ScrollableComponent, CustomerList };
}
```

```xml
<!-- static/src/list_view/list_view.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.ListViewComponent">
    <ScrollableComponent>
      <table class="table">
        <thead class="sticky-top bg-dark text-white">
          <tr><th>#</th><th>Image</th><th>Name</th><th>Email</th></tr>
        </thead>
        <tbody>
          <t t-foreach="props.partners" t-as="partner" t-key="partner.id">
            <CustomerList partner="partner" index="partner_index+1"/>
          </t>
        </tbody>
      </table>
    </ScrollableComponent>
  </t>
</templates>
```

### GridViewComponent wrapper

```javascript
// static/src/grid_view/grid_view.js
/** @odoo-module **/
import { Component } from "@odoo/owl";
import { ScrollableComponent } from "../standalone_app/components/scrollable_component/scrollable_component";
import { CustomerList } from "../customer_list/customer_list";

export class GridViewComponent extends Component {
  static template = "ica_movie.GridViewComponent";
  static props = {};
  static components = { ScrollableComponent, CustomerList };
}
```

```xml
<!-- static/src/grid_view/grid_view.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.GridViewComponent">
    <ScrollableComponent>
      <div class="row row-cols-1 row-cols-md-3 g-4">
        <t t-foreach="props.partners" t-as="partner" t-key="partner.id">
          <div class="col">
            <div class="card h-100">
              <img t-att-src="partner.imageURL" class="card-img-top" t-att-alt="partner.id"/>
              <div class="card-body">
                <h5 class="card-title"><t t-esc="partner.name"/></h5>
                <p class="card-text"><t t-esc="partner.email"/></p>
              </div>
            </div>
          </div>
        </t>
      </div>
    </ScrollableComponent>
  </t>
</templates>
```

#### Explanation of list/grid

- First, we define `CustomerList` to render a single row.  
- Next, we wrap the table or grid in `ScrollableComponent` for scroll support.  
- Then, the root `Customers` component (below) switches between these view modes.

## build customers screen – switch list/grid

First, build custom client action guides you to a `Customers` screen under `static/src/customers`:

```javascript
// static/src/customers/customers.js
/** @odoo-module **/
import { Component, onWillStart, useState } from "@odoo/owl";
import { ListViewComponent } from "../list_view/list_view";
import { GridViewComponent } from "../grid_view/grid_view";
import { registry } from "@web/core/registry";

const VIEW = { listView: "list", gridView: "grid" };

export class Customers extends Component {
  static template = "ica_movie.Customers";
  static props = {};
  static components = { ListViewComponent, GridViewComponent };

  setup() {
    this.state = useState({ view: VIEW.listView, partners: [] });
    this.orm = this.env.services.orm;
    onWillStart(async () => { await this.loadAll(); });
  }

  switchView() {
    this.state.view =
      this.state.view === VIEW.listView ? VIEW.gridView : VIEW.listView;
  }

  getComponent() {
    return this.state.view === VIEW.listView
      ? GridViewComponent
      : ListViewComponent;
  }

  async loadAll() {
    this.state.partners = (
      await this.orm.searchRead("res.partner", [], ["id", "name", "email"], { order: "id desc" })
    ).map((p) => ({
      ...p,
      imageURL: `/web/image/res.partner/${p.id}/avatar_128`,
    }));
  }
}

registry.category("ica.movie").add("customerScreen", Customers);
```

```xml
<!-- static/src/customers/customers.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Customers">
    <div class="d-flex justify-content-end m-3">
      <button class="btn bg-success text-white me-2 #{state.view==='list'?'disabled':''}" t-on-click="switchView">
        <i class="oi oi-view-list"/>
      </button>
      <button class="btn bg-success text-white #{state.view==='grid'?'disabled':''}" t-on-click="switchView">
        <i class="oi oi-view-kanban"/>
      </button>
    </div>
    <t t-component="getComponent()" partners="state.partners"/>
  </t>
</templates>
```

#### Explanation of customers screen

- First, we track `state.view` to switch modes.  
- Next, we load all partners on `onWillStart`.  
- Then, we render either the list or grid component dynamically.  
- Finally, we register this screen under `ica.movie` for the standalone app.

## build sale order screen – example registry

First, build custom client action can illustrate custom screens. Next, we add `SaleOrders`:

```javascript
// static/src/sale_orders/sale_orders.js
/** @odoo-module **/
import { Component, useState } from "@odoo/owl";
import { registry } from "@web/core/registry";

export class SaleOrders extends Component {
  static template = "ica_movie.SaleOrders";

  setup() {
    this.state = useState({ translateText: "ICA" });
  }

  getClick() {
    console.log("Button clicked");
  }
}

registry.category("ica.movie").add("saleOrderScreen", SaleOrders);
```

```xml
<!-- static/src/sale_orders/sale_orders.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.SaleOrders">
    <div class="container mt-5">
      <h3>Hello Sale Orders</h3>
      <div class="m-5">
        <h4><t t-esc="state.translateText"/></h4>
        <button class="btn btn-primary" t-on-click="getClick">Click Me</button>
      </div>
      <t t-call="ica_movie.so_sub_temp">
        <t t-set="translateText" t-value="state.translateText"/>
      </t>
    </div>
    <t t-name="ica_movie.so_sub_temp">
      <h4>Sub‑Template</h4>
      <h4><t t-esc="translateText"/></h4>
    </t>
  </t>
</templates>
```

#### Explanation of sale orders screen

- First, we use `useState` to handle local text.  
- Next, we register `saleOrderScreen` under the same registry category.  
- Then, `Root` (below) can render this screen when selected.

## build standalone OWL app – public page

First, build custom client action includes a public page. Next, we create `views/template.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
  <template id="ica_movie.standalone_app">
    <!DOCTYPE html>
    <html>
      <head>
        <script type="text/javascript">
          var odoo = {
            csrf_token: "<t t-esc="request.csrf_token(None)"/>",
            debug: "<t t-out="debug"/>",
            __session_info__: <t t-esc="json.dumps(session_info)"/>
          };
        </script>
        <t t-call-assets="ica_movie.assets_standalone_app"/>
      </head>
      <body/>
    </html>
  </template>
</odoo>
```

#### Explanation of standalone template

- First, we expose CSRF and session data in a global `odoo` object.  
- Next, we call the standalone asset bundle.  
- Then, the `<body/>` remains empty for OWL to mount.

## mount standalone OWL app – entry and root

First, build custom client action needs an entry point:

```javascript
// static/src/standalone_app/app.js
/** @odoo-module **/
import { whenReady } from "@odoo/owl";
import { mountComponent } from "@web/env";
import { Root } from "./root";

whenReady(() => mountComponent(Root, document.body));
```

```javascript
// static/src/standalone_app/root.js
/** @odoo-module **/
import { Component, useState, useSubEnv } from "@odoo/owl";
import { Navbar } from "./components/navbar/navbar";
import { registry } from "@web/core/registry";
import { createTodoStore } from "./todo_app/todo";

export class Root extends Component {
  static template = "ica_movie.Root";
  static components = { Navbar };

  setup() {
    this.state = useState({ mainScreen: "todo_list" });
    useSubEnv({
      switchScreen: this.switchScreen.bind(this),
      store: createTodoStore(),
    });
  }

  switchScreen(name) {
    this.state.mainScreen = name;
  }

  getComponent() {
    return registry.category("ica.movie").get(this.state.mainScreen);
  }
}
```

```xml
<!-- static/src/standalone_app/root.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Root">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js"/>
    <Navbar switchScreen.bind="switchScreen" mainScreen="state.mainScreen"/>
    <t t-call-assets="ica_movie.assets_standalone_app"/>
    <t t-component="getComponent()"/>
  </t>
</templates>
```

#### Explanation of entry and root

- First, `app.js` waits for DOM ready and mounts the `Root` component.  
- Next, `Root.setup()` initializes `mainScreen` and injects a Todo store.  
- Then, `Navbar` allows switching screens via `switchScreen`.  
- Finally, `getComponent()` renders the selected screen.

## build navbar and scrollable wrapper

First, build custom client action standalone UI includes navigation and scroll containers.

```javascript
// static/src/standalone_app/components/navbar/navbar.js
/** @odoo-module **/
import { Component } from "@odoo/owl";

export class Navbar extends Component {
  static template = "ica_movie.Navbar";
}
```

```xml
<!-- static/src/standalone_app/components/navbar/navbar.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Navbar">
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
      <div class="container-fluid">
        <a class="navbar-brand" href="#" t-on-click="() => env.switchScreen('customerScreen')">ICA</a>
        <div class="collapse navbar-collapse">
          <ul class="navbar-nav me-auto">
            <li class="nav-item" t-on-click="() => env.switchScreen('customerScreen')">
              <a class="nav-link" t-attf-class="#{props.mainScreen==='customerScreen'?'active':''}" href="#">Home</a>
            </li>
            <li class="nav-item" t-on-click="() => env.switchScreen('saleOrderScreen')">
              <a class="nav-link" t-attf-class="#{props.mainScreen==='saleOrderScreen'?'active':''}" href="#">Sale Orders</a>
            </li>
          </ul>
        </div>
      </div>
    </nav>
  </t>
</templates>
```

```javascript
// static/src/standalone_app/components/scrollable_component/scrollable_component.js
/** @odoo-module **/
import { Component } from "@odoo/owl";

export class ScrollableComponent extends Component {
  static template = "ica_movie.ScrollableComponent";
  static defaultProps = { class: "m-3 bg-white" };
}
```

```xml
<!-- static/src/standalone_app/components/scrollable_component/scrollable_component.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.ScrollableComponent">
    <div t-att-class="props.class" style="max-height:80vh;overflow-y:auto;">
      <t t-slot="default"/>
    </div>
  </t>
</templates>
```

#### Explanation of navbar and wrapper

- First, `Navbar` uses Bootstrap for responsive navigation.  
- Next, it calls `env.switchScreen()` to change the root screen.  
- Then, `ScrollableComponent` wraps content in a scrollable container.  
- Finally, we apply default props for consistent styling.

## build Todo app in standalone

First, build custom client action standalone page can host a Todo app. Next, we create a reactive store:

```javascript
// static/src/standalone_app/todo_app/todo.js
/** @odoo-module **/
import { reactive } from "@odoo/owl";

class Todo {
  nextId = 3;
  todos = [{ id: 1, name: "hello" }, { id: 2, name: "hello 2" }];

  addTask(task) {
    task.id = this.nextId++;
    this.todos.push(task);
  }

  deleteTask(id) {
    this.todos = this.todos.filter((t) => t.id !== id);
  }
}

export function createTodoStore() {
  return reactive(new Todo());
}
```

Next, we build `TodoList` and `TodoItem`:

```javascript
// static/src/standalone_app/todo_app/todo_list.js
/** @odoo-module **/
import { Component } from "@odoo/owl";
import { useAutofocus } from "@web/core/utils/hooks";
import { useStore } from "./todo";

export class TodoList extends Component {
  static template = "ica_movie.TodoList";

  setup() {
    this.inputRef = useAutofocus({ refName: "title" });
    this.store = useStore();
  }

  addTask() {
    this.store.addTask({ name: this.inputRef.el.value });
  }
}

registry.category("ica.movie").add("todo_list", TodoList);
```

```xml
<!-- static/src/standalone_app/todo_app/todo_list.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.TodoList">
    <h3>Todo List</h3>
    <div class="d-flex mb-3">
      <input t-ref="title" type="text" class="form-control me-2" placeholder="Title"/>
      <button class="btn btn-warning" t-on-click="addTask">Add Task</button>
    </div>
    <t t-foreach="env.store.todos" t-as="todo" t-key="todo.id">
      <t t-call="ica_movie.TodoItem">
        <t t-set="todo" t-value="todo"/>
      </t>
    </t>
  </t>

  <t t-name="ica_movie.TodoItem">
    <link rel="stylesheet" href="/ica_movie/static/src/standalone_app/todo_app/style.scss"/>
    <div class="card mb-2">
      <div class="card-body d-flex justify-content-between">
        <span><t t-esc="todo.name"/></span>
        <button class="btn btn-danger" t-on-click="() => env.store.deleteTask(todo.id)">Delete</button>
      </div>
    </div>
  </t>
</templates>
```

```scss
/* static/src/standalone_app/todo_app/style.scss */
button.btn-warning {
  background-color: #000 !important;
  color: #fff !important;
}
```

#### Explanation of Todo app

- First, `createTodoStore()` returns a reactive store with `todos`.  
- Next, `TodoList` uses `useStore()` and `useAutofocus()` for input focus.  
- Then, it renders each `TodoItem` and allows deletion.  
- Finally, we include a custom SCSS file for button styling.

## register actions and menus

First, build custom client action needs menu entries in `views/ica_movie_client_action.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
  <record model="ir.actions.client" id="ica_movie_action">
    <field name="name">Movies</field>
    <field name="tag">ica_movie.movieAction</field>
  </record>

  <record model="ir.actions.act_url" id="ica_standalone_action">
    <field name="name">Standalone</field>
    <field name="url">/ica-movie/standalone_app</field>
    <field name="target">self</field>
  </record>

  <menuitem id="ica_movie_root" name="ICA"/>
  <menuitem id="movie_category" name="Movie" parent="ica_movie_root" action="ica_movie_action"/>
  <menuitem id="standalone_category" name="Standalone" parent="ica_movie_root" action="ica_standalone_action"/>
</odoo>
```

#### Explanation of menus

- First, `ir.actions.client` launches the back‑end OWL action.  
- Next, `ir.actions.act_url` opens the public standalone page.  
- Then, we group both under a root menu “ICA.”  
- Finally, users see “Movie” and “Standalone” options in the top bar.

## test and deploy your custom client action

First, build custom client action by restarting Odoo with `-u ica_movie`. Next, enable developer mode in your database. Then, click **ICA > Movie** to launch the OWL client action. Moreover, test theme toggling, real‑time messages, partner CRUD, RPC login, HTTP fetch, router toggling, and CP hiding. After that, click **ICA > Standalone** to open the public page. Finally, switch between Customers, Sale Orders, and Todo List screens.

## conclusion

First, build custom client action in Odoo 17 OWL empowers you to create reactive, modular, and real‑time UI components. Next, you mastered manifest updates, HTTP routes, model extensions, OWL component setup, QWeb templates, and lifecycle patches. Then, you built list/grid view modes, sale order examples, and a standalone app with navigation and a Todo store. Finally, you learned to register menus, test features, and link to external resources like [JSONPlaceholder](https://jsonplaceholder.typicode.com) for HTTP services. Moreover, you can adapt these patterns to develop robust OWL client actions for any domain.