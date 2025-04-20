# ica_owl_batch_1

This document presents the `ica_movie` add‑on sources, reorganized into proper Markdown with file sections and fenced code blocks.

## 1. controllers/main.py

```python
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
        # data = {
        #     'session_info': {'user_context': {'lang': 'my_MM'}},
        # }
        return request.render(
            'ica_movie.standalone_app',
            get_frontend_session_info
        )
```

## 2. models/res_partner.py

```python
from odoo import api, fields, models

class Respartner(models.Model):
    _inherit = 'res.partner'

    def action_class_from_json(self, name, email):
        print("*" * 100)
        print(self)
        print(name, email)
```

## 3. static/src/customer_list/customer_list.js

```javascript
/** @odoo-module */
import { Component, onWillStart, useState } from "@odoo/owl";

export class CustomerList extends Component {
    static template = "ica_movie.CustomerList";
    static props = {};

    // setup(){
    //    console.log(this.props)
    // }
}
```

## 4. static/src/customer_list/customer_list.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.CustomerList">
    <t t-set="partner" t-value="props.partner"/>
    <tr>
      <th scope="row"><span t-esc="props.index"/></th>
      <td>
        <t t-if="partner.imageURL">
          <img t-att-src="partner.imageURL"/>
        </t>
        <t t-else="">
          <span>-</span>
        </t>
      </td>
      <td>
        <t t-if="partner.name">
          <span t-esc="partner.name"/>
        </t>
        <t t-else="">
          <span>-</span>
        </t>
      </td>
      <td>
        <t t-if="partner.email">
          <span t-esc="partner.email"/>
        </t>
        <t t-else="">
          <span>-</span>
        </t>
      </td>
    </tr>
  </t>
</templates>
```

## 5. static/src/customers/customers.js

```javascript
/** @odoo-module */
import { Component, onWillStart, useState } from "@odoo/owl";
import { CustomerList } from "../customer_list/customer_list";
import { ScrollableComponent } from "../standalone_app/components/scrollable_component/scrollable_component";
import { ListViewComponent } from "../list_view/list_view";
import { GridViewComponent } from "../grid_view/grid_view";
import { registry } from "@web/core/registry";

const VIEW = {
    listView: "list",
    gridView: "grid",
};

export class Customers extends Component {
    static template = "ica_movie.Customers";
    static props = {};
    static components = { ListViewComponent, GridViewComponent };

    setup() {
        this.state = useState({ view: VIEW.listView, partners: [] });
        this.model = "res.partner";
        this.orm = this.env.services.orm;

        onWillStart(async () => {
            await this.getAllPartners();
        });
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

    async getAllPartners() {
        this.state.partners = await this.orm.searchRead(
            this.model,
            [],
            ["name", "email"]
        );
        this.state.partners = this.state.partners.map((partner) => ({
            ...partner,
            imageURL: `/web/image/${this.model}/${partner.id}/avatar_128`,
        }));
    }
}

registry.category("ica.movie").add("customerScreen", Customers);
```

## 6. static/src/customers/customers.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Customers">
    <div class="mx-5 mt-5 mb-2 d-flex justify-content-end">
      <button
        t-attf-class="btn bg-success text-white mx-2 #{state.view === 'list' ? 'disabled' : ''}"
        t-on-click="switchView"
      >
        <i class="oi oi-view-list"/>
      </button>
      <button
        t-attf-class="#{state.view === 'grid' ? 'disabled' : ''}"
        class="btn bg-success text-white"
        t-on-click="switchView"
      >
        <i class="oi oi-view-kanban"/>
      </button>
    </div>
    <t t-component="getComponent()" partners="state.partners"/>
  </t>
</templates>
```

## 7. static/src/grid_view/grid_view.js

```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
import { ScrollableComponent } from "../standalone_app/components/scrollable_component/scrollable_component";
import { CustomerList } from "../customer_list/customer_list";

export class GridViewComponent extends Component {
    static template = "ica_movie.GridViewComponent";
    static props = {};
    static components = { ScrollableComponent, CustomerList };
}
```

## 8. static/src/grid_view/grid_view.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.GridViewComponent">
    <ScrollableComponent>
      <t t-set-slot="title"/>
      <div class="row row-cols-1 row-cols-md-3 g-4">
        <t t-foreach="props.partners" t-as="partner" t-key="partner.id">
          <div class="col">
            <div class="card h-100">
              <img
                t-att-src="partner.imageURL"
                class="card-img-top"
                t-att-alt="partner.id"
              />
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

*(Additional files such as `ica_movie.js`, `ica_movie.xml`, `IcaMovieActionER.js`, standalone app code, views, and SCSS follow the same pattern: each file under `static/src` or `views/` is given its own section with fenced code blocks.)*

### ica_movie/static/src/ica_movie/ica_movie.js
```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { Component, useState, onWillStart, useRef } from "@odoo/owl";
import { ConfirmationDialog } from "@web/core/confirmation_dialog/confirmation_dialog";
import { Layout } from "@web/search/layout";
import { Notebook } from "@web/core/notebook/notebook";
import { useService } from "@web/core/utils/hooks";
import { cookie } from "@web/core/browser/cookie";
import { browser } from "@web/core/browser/browser";
import { routeToUrl } from "@web/core/browser/router_service";
import { useAutofocus } from "@web/core/utils/hooks";

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
    this.dialog = this.env.services.dialog;
    this.effectService = this.env.services.effect;
    this.httpService = this.env.services.http;
    this.userService = this.env.services.user;
    this.busService = this.env.services.bus_service;
    this.rpcService = this.env.services.rpc;

    this.busService.addChannel("ica-movie-channel");
    this.busService.subscribe(
      "ica-movie-channel/sending-message",
      (payload) => this.state.messages.push(payload.message)
    );

    onWillStart(async () => {
      await this.getAllPartners();
      this.changeTitle();
      this.state.darkTheme = cookie.get("darkTheme") === "true";
      this.getUser();
    });
  }

  async sendMessage() {
    const message = this.inputRef.el.value.trim();
    if (message) {
      await this.rpcService("/ica/send-bus", { message });
      this.inputRef.el.value = "";
    }
  }

  async searchPartners(e) {
    if (e.type === "click" || e.keyCode === 13) {
      const name = this.nameRef.el.value.trim();
      await this.getAllPartners(name);
      this.nameRef.el.value = "";
    }
  }

  async getAllPartners(name = "") {
    this.state.partners = await this.orm.searchRead(
      this.resModel,
      [["name", "ilike", name]],
      ["id", "name", "email", "phone"],
      { order: "id desc" }
    );
  }

  deletePartner(partner) {
    this.dialog.add(
      ConfirmationDialog,
      {
        title: "Delete",
        body: `Are you sure to delete ${partner.name}?`,
        confirm: async () => {
          await this.orm.unlink(this.resModel, [partner.id]);
          this.state.partners = this.state.partners.filter(
            (p) => p.id !== partner.id
          );
          this.effectService.add({
            type: "rainbow_man",
            message: "Record deleted successfully.",
          });
        },
      },
      { onClose: () => console.log("Dialog closed") }
    );
  }

  async updatePartner(partner) {
    this.state.partner = partner;
    this.state.activeId = partner.id;
  }

  async savePartner() {
    if (this.state.activeId) {
      await this.orm.write(
        this.resModel,
        [this.state.activeId],
        this.state.partner
      );
      this.effectService.add({
        type: "notification",
        message: "Partner updated",
      });
      this.state.activeId = null;
    } else {
      const [id] = await this.orm.create(this.resModel, [this.state.partner]);
      this.state.partners.push({ ...this.state.partner, id });
    }
    this.state.partner = { name: "", email: "", phone: "" };
  }

  async getTodoList() {
    this.state.todos = await this.httpService.get(
      "https://jsonplaceholder.typicode.com/todos"
    );
  }

  changeTitle() {
    this.env.services.title.setParts({ zopenerp: "Movies" });
  }

  switchTheme() {
    const next = cookie.get("darkTheme") !== "true";
    cookie.set("darkTheme", next);
    this.state.darkTheme = next;
  }

  getUser() {
    const uid = this.userService.partnerId;
    this.state.image = `/web/image?model=res.partner&id=${uid}&field=avatar_128`;
  }

  changeRouter() {
    const { current } = this.env.services.router;
    current.search.debug = !current.search.debug;
    current.search.darkTheme = !current.search.darkTheme;
    browser.location.href =
      browser.location.origin + routeToUrl(current);
  }

  getCompany() {
    this.display.controlPanel.topRight = false;
  }
}

registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);
```

### ica_movie/static/src/ica_movie/ica_movie.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.icaMovie">
    <div t-att-class="{{ state.darkTheme ? 'bg-dark text-white' : '' }}">
      <Layout display="display">
        <div class="container d-flex justify-content-between mt-5">
          <button class="btn btn-primary" t-on-click="switchTheme">
            Switch Theme
          </button>
          <img t-att-src="state.image" class="rounded-circle" width="40"/>
        </div>
        <div class="container mt-5">
          <Notebook orientation="'horizontal'">
            <t t-set-slot="page_0" title="'Messages'" isVisible="true">
              <h3>Messages</h3>
              <div style="max-height:200px;overflow-y:auto;">
                <t t-foreach="state.messages" t-as="msg" t-key="msg_index">
                  <div class="alert alert-info m-2"><t t-esc="msg"/></div>
                </t>
              </div>
              <div class="d-flex mt-3">
                <input
                  t-ref="input-box"
                  class="form-control me-2"
                  placeholder="type a message..."
                />
                <button class="btn btn-success" t-on-click="sendMessage">
                  Send
                </button>
              </div>
            </t>

            <t t-set-slot="page_1" title="'Movies'" isVisible="true">
              <h3>Movie Partners</h3>
              <div class="d-flex mb-3">
                <button
                  class="btn btn-outline-primary me-2 oi oi-search"
                  t-on-click="searchPartners"
                />
                <button
                  class="btn btn-primary"
                  data-bs-toggle="modal"
                  data-bs-target="#partnerModal"
                >
                  New Partner
                </button>
              </div>
              <div style="max-height:400px;overflow-y:auto;">
                <table class="table table-striped">
                  <thead class="table-dark">
                    <tr>
                      <th>#</th><th>Name</th><th>Phone</th><th>Email</th><th>Actions</th>
                    </tr>
                  </thead>
                  <tbody>
                    <t t-foreach="state.partners" t-as="p" t-key="p.id">
                      <tr>
                        <td><t t-esc="p_index+1"/></td>
                        <td><t t-esc="p.name||'-'"/></td>
                        <td><t t-esc="p.phone||'-'"/></td>
                        <td><t t-esc="p.email||'-'"/></td>
                        <td>
                          <button
                            class="btn btn-sm btn-warning me-1"
                            t-on-click="() => updatePartner(p)"
                            data-bs-toggle="modal"
                            data-bs-target="#partnerModal"
                          >Edit</button>
                          <button
                            class="btn btn-sm btn-danger me-1"
                            t-on-click="() => deletePartner(p)"
                          >Delete</button>
                        </td>
                      </tr>
                    </t>
                  </tbody>
                </table>
              </div>

              <div
                class="modal fade"
                id="partnerModal"
                tabindex="-1"
                aria-hidden="true"
              >
                <div class="modal-dialog">
                  <div class="modal-content">
                    <div class="modal-header">
                      <h5 class="modal-title">Partner Form</h5>
                      <button
                        type="button"
                        class="btn-close"
                        data-bs-dismiss="modal"
                      />
                    </div>
                    <div class="modal-body">
                      <input
                        type="text"
                        class="form-control mb-2"
                        placeholder="Name"
                        t-model="state.partner.name"
                      />
                      <input
                        type="text"
                        class="form-control mb-2"
                        placeholder="Email"
                        t-model="state.partner.email"
                      />
                      <input
                        type="text"
                        class="form-control mb-2"
                        placeholder="Phone"
                        t-model="state.partner.phone"
                      />
                    </div>
                    <div class="modal-footer">
                      <button
                        type="button"
                        class="btn btn-secondary"
                        data-bs-dismiss="modal"
                      >
                        Close
                      </button>
                      <button
                        type="button"
                        class="btn btn-primary"
                        data-bs-dismiss="modal"
                        t-on-click="savePartner"
                      >
                        Save
                      </button>
                    </div>
                  </div>
                </div>
              </div>
            </t>

            <t t-set-slot="page_2" title="'HTTP Service'" isVisible="true">
              <h3>Load Todo List</h3>
              <button
                class="btn btn-outline-secondary mb-3"
                t-on-click="getTodoList"
              >
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
          </Notebook>
        </div>
      </Layout>
    </div>
  </t>
</templates>
```

### ica_movie/static/src/ica_movie/IcaMovieActionER.js
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

### ica_movie/static/src/list_view/list_view.js
```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
import { ScrollableComponent } from "../standalone_app/components/scrollable_component/scrollable_component";
import { CustomerList } from "../customer_list/customer_list";

export class ListViewComponent extends Component {
  static template = "ica_movie.ListViewComponent";
  static props = {};
  static components = { ScrollableComponent, CustomerList };
}
```

### ica_movie/static/src/list_view/list_view.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.ListViewComponent">
    <ScrollableComponent>
      <table class="table">
        <thead class="sticky-top bg-dark text-white">
          <tr>
            <th>#</th><th>Image</th><th>Name</th><th>Email</th>
          </tr>
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

### ica_movie/static/src/sale_orders/sale_orders.js
```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
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

### ica_movie/static/src/sale_orders/sale_orders.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.SaleOrders">
    <div class="container mt-5">
      <h3>Hello Sale Orders</h3>
      <div class="m-5">
        <h4><t t-esc="state.translateText"/></h4>
        <button class="btn btn-primary" t-on-click="getClick">Click Me</button>
      </div>
    </div>
    <t t-call="ica_movie.so_sub_temp">
      <t t-set="translateText" t-value="state.translateText"/>
    </t>
  </t>

  <t t-name="ica_movie.so_sub_temp">
    <h4>Sub‑Template</h4>
    <h4><t t-esc="translateText"/></h4>
  </t>
</templates>
```

### ica_movie/static/src/standalone_app/app.js
```javascript
/** @odoo-module */
import { whenReady } from "@odoo/owl";
import { mountComponent } from "@web/env";
import { Root } from "./root";

whenReady(() => mountComponent(Root, document.body));
```

### ica_movie/static/src/standalone_app/components/navbar/navbar.js
```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";

export class Navbar extends Component {
  static template = "ica_movie.Navbar";
}
```

### ica_movie/static/src/standalone_app/components/navbar/navbar.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Navbar">
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
      <div class="container-fluid">
        <a
          class="navbar-brand"
          href="#"
          t-on-click="() => env.switchScreen('customerScreen')"
        >ICA</a>
        <div class="collapse navbar-collapse">
          <ul class="navbar-nav me-auto">
            <li class="nav-item" t-on-click="() => env.switchScreen('customerScreen')">
              <a
                class="nav-link"
                t-attf-class="#{props.mainScreen==='customerScreen'?'active':''}"
                href="#"
              >Home</a>
            </li>
            <li class="nav-item" t-on-click="() => env.switchScreen('saleOrderScreen')">
              <a
                class="nav-link"
                t-attf-class="#{props.mainScreen==='saleOrderScreen'?'active':''}"
                href="#"
              >Sale Orders</a>
            </li>
          </ul>
        </div>
      </div>
    </nav>
  </t>
</templates>
```

### ica_movie/static/src/standalone_app/components/scrollable_component/scrollable_component.js
```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";

export class ScrollableComponent extends Component {
  static template = "ica_movie.ScrollableComponent";
  static defaultProps = { class: "m-3 bg-white" };
}
```

### ica_movie/static/src/standalone_app/components/scrollable_component/scrollable_component.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.ScrollableComponent">
    <div t-att-class="props.class" style="max-height:80vh;overflow-y:auto;">
      <t t-slot="default"/>
    </div>
  </t>
</templates>
```

### ica_movie/static/src/standalone_app/root.js
```javascript
/** @odoo-module */
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

### ica_movie/static/src/standalone_app/root.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
  <t t-name="ica_movie.Root">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js"/>
    <Navbar switchScreen.bind="switchScreen" mainScreen="state.mainScreen"/>
    <t t-component="getComponent()"/>
  </t>
</templates>
```

### ica_movie/static/src/standalone_app/todo_app/style.scss
```scss
button.btn-warning {
  background-color: #000 !important;
  color: #fff !important;
}
```

### ica_movie/static/src/standalone_app/todo_app/todo.js
```javascript
/** @odoo-module */
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

export function useStore() {
  const env = useEnv();
  return useState(env.store);
}
```

### ica_movie/static/src/standalone_app/todo_app/todo_list.js
```javascript
/** @odoo-module */
import { Component, useAutofocus } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useStore } from "./todo";

export class TodoList extends Component {
  static template = "ica_movie.TodoList";

  setup() {
    this.inputRef = useAutofocus({ refName: "title" });
    this.store = useStore();
  }

  addTask() {
    this.store.addTask({ name: this.inputRef.el.value.trim() });
    this.inputRef.el.value = "";
  }
}

registry.category("ica.movie").add("todo_list", TodoList);
```

### ica_movie/static/src/standalone_app/todo_app/todo_list.xml
```xml
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
        <button
          class="btn btn-danger"
          t-on-click="() => env.store.deleteTask(todo.id)"
        >
          Delete
        </button>
      </div>
    </div>
  </t>
</templates>
```

### ica_movie/views/ica_movie_client_action.xml
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
  </record>

  <menuitem id="ica_movie_root" name="ICA">
    <menuitem
      id="movie_category"
      name="Movie"
      action="ica_movie_action"
      sequence="0"
    />
    <menuitem
      id="standalone_category"
      name="Standalone"
      action="ica_standalone_action"
      sequence="1"
    />
  </menuitem>
</odoo>
```

### ica_movie/views/template.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
  <template id="ica_movie.standalone_app">
    <!DOCTYPE html>
    <html>
      <head>
        <script type="text/javascript">
          var odoo = {
            csrf_token: "<t t-esc='request.csrf_token(None)'/>",
            debug: "<t t-out='debug'/>",
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