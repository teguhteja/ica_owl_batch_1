# ica_owl_batch_1

"""
/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/controllers/main.py
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
            'ica-movie-channel/sending-message', kw)
        return True

    @http.route("/ica-movie/standalone_app", auth="public")
    def standalone_app(self, **kw):
        get_frontend_session_info: dict = request.env['ir.http'].session_info()
        # data = {
        #     'session_info': {'user_context': {'lang': 'my_MM', }},
        # }
        return request.render(
            'ica_movie.standalone_app', get_frontend_session_info
        )

/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/models/res_partner.py
from odoo import api, fields, models


class Respartner(models.Model):
    _inherit = 'res.partner'

    def action_class_from_json(self,name,email):
        print("*" * 100)
        print(self)
        print(name,email)

/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/customer_list/customer_list.js
/** @odoo-module */
import {Component,onWillStart,useState} from "@odoo/owl";

export class CustomerList extends Component {
    static template = "ica_movie.CustomerList";
    static props = {};

    // setup(){
    //     console.log(this.props)
    // }
}

/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/customer_list/customer_list.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.CustomerList">
        <t t-set="partner" t-value="props.partner"/>
        <tr>
            <th scope="row"><span t-esc="props.index"/></th>
            <td>
                <t t-if="partner.imageURL">

                   <img t-att-src="partner.imageURL"/>
<!--                   <img t-attf-src="/web/image/res.partner/{{partner.id}}/avatar_128"/>-->
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


/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/customers/customers.js
/** @odoo-module */
import {Component, onWillStart, useState} from "@odoo/owl";
import {CustomerList} from "../customer_list/customer_list";
import {ScrollableComponent} from "../standalone_app/components/scrollable_component/scrollable_component";
import {ListViewComponent} from "../list_view/list_view";
import {GridViewComponent} from "../grid_view/grid_view";
import { registry } from "@web/core/registry";

const VIEW = {
    listView: "list",
    gridView: "grid",
}

export class Customers extends Component {
    static template = "ica_movie.Customers";
    static props = {};
    static components = {ListViewComponent, GridViewComponent};

    setup() {
        this.state = useState({
            view: VIEW.listView,
            partners: []
        })
        this.model = "res.partner";
        this.orm = this.env.services.orm;

        onWillStart(async () => {
            await this.getAllPartners();
        })
    }

    switchView() {
        this.state.view = this.state.view === VIEW.listView ? VIEW.gridView : VIEW.listView;
    }

    getComponent() {
        return this.state.view === VIEW.listView ? GridViewComponent : ListViewComponent;
    }

    async getAllPartners() {
        this.state.partners = await this.orm.searchRead(this.model, [], ['name', 'email']);
        this.state.partners = this.state.partners.map(partner => {
            return {...partner, imageURL: `/web/image/${this.model}/${partner.id}/avatar_128`}
            // return {email:partner.email,name:partner.name, imageURL: `/web/image/${this.model}/${partner.id}/avatar_128`}
        })
    }
}

registry.category('ica.movie').add('customerScreen',Customers)

/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/customers/customers.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.Customers">
        <div class="mx-5 mt-5 mb-2 d-flex justify-content-end">
<!--            <t t-esc="state.view"/>-->
            <!--             t-if="state.view === 'list'"-->
            <!--            t-else="state.view === 'grid'"-->
            <!--            t-attf-class="badge rounded-pill o_tag o_tag_color_#{tag.color} d-inline-block"-->
            <button t-attf-class="btn bg-success text-white mx-2 #{state.view ==='list'?'disabled':''}"
                    t-on-click="switchView">
                <i class="oi oi-view-list"/>
            </button>
            <button t-attf-class="#{state.view ==='grid'?'disabled':''}"
                    class="btn bg-success text-white"
                    t-on-click="switchView">
                <i class="oi oi-view-kanban"/>
            </button>

        </div>
        <t t-component="getComponent()" partners="state.partners"/>

        <!--        <ListViewComponent partners="state.partners" t-if="state.view === 'list'"/>-->
        <!--        <GridViewComponent partners="state.partners" t-else=""/>-->
    </t>
        </templates>


/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/grid_view/grid_view.js
/** @odoo-module */
import {Component, onWillStart, useState} from "@odoo/owl";
import {ScrollableComponent} from "../standalone_app/components/scrollable_component/scrollable_component";
import {CustomerList} from "../customer_list/customer_list";

export class GridViewComponent extends Component {
    static template = "ica_movie.GridViewComponent";
    static props = {};
    static components = {ScrollableComponent, CustomerList}
}

/home/teguhteja/Project/OdooProjects/o17-p16-d03/custom-vf/ica_movie/static/src/grid_view/grid_view.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.GridViewComponent">

<!--        <ScrollableComponent  class="'bg-warning'">-->
        <ScrollableComponent>
            <t t-set-slot="title"/>
            <div class="row row-cols-1 row-cols-md-3 g-4">
                <t t-foreach="props.partners" t-as="partner" t-key="partner.id">
                  <div class="col">
                    <div class="card h-100">
                      <img t-att-src="partner.imageURL" class="card-img-top" t-att-alt="partner.id"/>
                        <div class="card-body">
                        <h5 class="card-title"><t t-esc="partner.name"/></h5>
                            <p class="card-text">
                                <t t-esc="partner.email"/>
                            </p>
                      </div>
                    </div>
                  </div>
                </t>
            </div>

        </ScrollableComponent>
    </t>
</templates>

custom-vf/ica_movie/static/src/ica_movie/ica_movie.js
/** @odoo-module **/

import {registry} from "@web/core/registry";
import {Component, useState, onWillStart, useRef} from "@odoo/owl";
import {ConfirmationDialog} from "@web/core/confirmation_dialog/confirmation_dialog";
import {Layout} from "@web/search/layout";
import {Notebook} from "@web/core/notebook/notebook";
import {useService} from "@web/core/utils/hooks";
import {cookie} from "@web/core/browser/cookie";
import {browser} from "@web/core/browser/browser";
import {routeToUrl} from "@web/core/browser/router_service";
import {useAutofocus} from "@web/core/utils/hooks";


export default class IcaMovieAction extends Component {
    static template = "ica_movie.icaMovie";
    static components = {Layout, Notebook};

    setup() {
        this.nameRef = useAutofocus({refName: 'name'});
        this.inputRef = useRef('input-box');
        // #useRef('name');
        this.display = {
            controlPanel: {topRight: true},
        }
        this.state = useState({
            partners: [],
            partner: {name: "", email: "", phone: ""},
            activeId: null,
            todos: [],
            darkTheme: false,
            image: null,
            messages: []
        })
        this.resModel = 'res.partner';
        this.orm = this.env.services.orm;
        this.dialog = this.env.services.dialog;
        this.effectService = this.env.services.effect;
        this.httpService = this.env.services.http;
        this.userService = this.env.services.user;
        this.busService = this.env.services.bus_service;
        this.rpcService = this.env.services.rpc;
        this.busService.addChannel('ica-movie-channel');
        this.busService.subscribe('ica-movie-channel/sending-message', payload => {
            this.state.messages.push(payload.message);
        });
        // this.busService.addEventListener('notification',payload=>{
        //     console.log(payload);
        // });
        // this.cookieService = useService("cookie");
        onWillStart(async () => {
            await this.getAllPartners();
            this.changeTitle();
            // console.log(cookie.get('darkTheme'));
            this.state.darkTheme = cookie.get('darkTheme') === 'true';
            this.getUser();
        })
    }

    async sendMessage() {
        var message = this.inputRef.el.value;
        console.log(message);
        // console.log(this.busService)
        await this.rpcService("/ica/send-bus", {
            "message": message
        });
        this.inputRef.el.value = '';
        // this.state.messages.push("Hello")
    }

    async searchPartners(e) {
        if (e.type === 'click' || e.keyCode === 13) {
            let name = this.nameRef.el.value.trim();
            await this.getAllPartners(name);
            this.nameRef.el.value = null;
            return null;
        }

    }

    async getAllPartners(name) {
        const domain = [['name', 'ilike', name]];
        const fields = ['id', 'name', 'email', 'phone'];
        this.state.partners = await this.orm.searchRead(this.resModel, domain, fields, {order: "id desc"});
    }

    deletePartner(newPartner) {
        this.dialog.add(ConfirmationDialog, {
                title: "Delete",
                body: `Are you sure to delete ${newPartner.name}?`,
                confirm: async () => {
                    await this.orm.unlink(this.resModel, [newPartner.id]);
                    this.state.partners = this.state.partners.filter(partner => partner.id !== newPartner.id);

                    //     effect
                    this.effectService.add({
                        type: "rainbow_man",
                        message: "Record Delete Successfully."
                    })
                },
                cancel: () => {
                },
            },
            {
                onClose: () => {
                    console.log("Helllo on Close.")
                }
            });
    }

    async updatePartner(newPartner) {
        this.state.partner = newPartner;
        this.state.activeId = newPartner.id;
    }

    async savePartner() {
        if (this.state.activeId) {
            await this.orm.write(this.resModel, [this.state.activeId],
                this.state.partner);
            this.state.activeId = false;
            this.notificationService = this.env.services.notification;
            this.notificationService.add("I'm a very simple notification", {
                title: "Title",
                type: "danger",
                sticky: true,
                className: "p-4",
                buttons: [
                    {
                        name: "Sample Button",
                        onClick: () => {
                            console.log("button click")
                        },
                        primary: true
                    },
                    {
                        name: "Sample Button 2",
                        onClick: () => {
                            console.log("button click")
                        },
                        primary: false
                    }
                ]
            });
        } else {
            var newPartner = await this.orm.create(this.resModel,
                [this.state.partner]);
            this.state.partners.push({
                ...this.state.partner,
                id: newPartner[0]
            });
        }
        this.state.partner = {};
    }

    async getTodoList() {
        const endPoint = 'https://jsonplaceholder.typicode.com/todos';
        var todos = await this.httpService.get(endPoint);
        console.log(todos);
        this.state.todos = todos;
    }

    changeTitle() {
        this.titleService = this.env.services.title;
        // console.log("Hello")
        this.titleService.setParts({
            // odoo: "AA",
            // fruit: "BB",
            zopenerp: "Change Action"
        });
        // console.log(this.titleService.current);
    }

    switchTheme() {
        cookie.get("darkTheme") === 'false'
            ? cookie.set("darkTheme", true)
            : cookie.set("darkTheme", false);
        this.state.darkTheme = cookie.get('darkTheme') === 'true';
    }

    getUser() {
        console.log(this.userService)
        this.state.image = `/web/image?model=res.partner&id=${this.userService.partnerId}&field=avatar_128&unique=1723729851000`;
    }

    changeRouter() {
        this.routerService = this.env.services.router;
        const {search} = this.routerService.current;
        search.debug = !search.debug;
        search.darkTheme = !search.darkTheme
        // console.log(this.routerService.current)
        browser.location.href = browser.location.origin +
            routeToUrl(this.routerService.current);
    }

    getCompany() {
        this.companyService = this.env.services.company;
        // console.log(this.companyService.currentCompany)
        // console.log(this.companyService.currency)
        // console.log(this.display.controlPanel.topRight)
        this.display.controlPanel.topRight = false;
        // console.log(this.display.controlPanel.topRight)
    }


}


// IcaMovieAction.template = "ica_movie.icaMovie";
// IcaMovieAction.components = {Layout,};

// remember the tag name we put in the first step
registry.category("actions").add("ica_movie.movieAction", IcaMovieAction);

custom-vf/ica_movie/static/src/ica_movie/ica_movie.xml
<?xml version="1.0" encoding="UTF-8" ?>
<templates xml:space="preserve">
    <t t-name="ica_movie.icaMovie">
        <div t-attf-class="{{state.darkTheme? 'bg-dark text-white':''}}">
             <Layout display="display">

                 <div class="container d-flex justify-content-between mt-5">
                <div class="container">
                <h1><t t-esc="state.darkTheme"/></h1>
                    <button class="btn btn-primary" t-on-click="switchTheme">Switch Theme</button>
            </div>
                     <div class="container">
                     <img t-att-src="state.image"/>
                 </div>
            </div>
                 <div class="container mt-5">
                <Notebook orientation="'horizontal'">
                     <t t-set-slot="page_0" title="'Messages'" isVisible="true">
                         <div class="container mt-5">
                             <h1>Messages</h1>
                             <div style="max-height: 200px; overflow-y: auto;">
                                 <t t-foreach="state.messages" t-as="data" t-key="data_index">
                                     <div class="container m-2 bg-dark text-white">
                                         <p><t t-esc="data"/></p>
                                     </div>
                                 </t>
                            </div>
                             <div class="d-flex">
                                 <input t-ref="input-box" class="form-control m-5" placeholder="type a message..."/>
                                 <button class="btn btn-primary mt-5"
                                         t-on-click="sendMessage">sendMessage</button>
                             </div>

                </div>
                     </t>
                    <t t-set-slot="page_1" title="'Movies'" isVisible="true">
                    <div class="container">
                                    <h1 class="mt-5 mb-5">Welcome From Movies!</h1>
                        <div class="d-flex justify-content-between mb-2 mt-2">
<!--                            <input t-ref="name" type="text" class="form-control" t-on-keyup="(e)=>this.searchPartners(e)"/>-->
<!--                            <input t-ref="name" type="text" class="form-control" t-on-key/>-->
                            <button class="btn btn-primary oi oi-search mx-2" t-on-click="searchPartners"/>
                            <!-- Button trigger modal -->
                            <button type="button" class="btn btn-primary mx-2" data-bs-toggle="modal"
                                    data-bs-target="#exampleModal">
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
                                                        <th scope="row"><span t-esc="partner_index+1"/></th>
                                                        <td>
                                                            <t t-if="partner.name">
                                                                <span t-esc="partner.name"/>
                                                            </t>
                                                            <t t-else="">
                                                                <span>-</span>
                                                            </t>
                                                        </td>
                                                        <td>
                                                            <t t-if="partner.phone">
                                                                <span t-esc="partner.phone"/>
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
                                                        <td>
                                                            <button class="btn btn-warning m-2"
                                                                    t-on-click="()=>this.updatePartner(partner)"
                                                                    data-bs-toggle="modal"
                                                                    data-bs-target="#exampleModal">Update</button>
                                                            <button class="btn btn-danger m-2"
                                                                    t-on-click="()=>this.deletePartner(partner)">Delete</button>
                                                            <button class="btn btn-primary m-2"
                                                                    t-on-click="()=>this.callOrmMethod(partner)">Call Method</button>
                                                        </td>
                                                    </tr>
                                                </t>
                                            </tbody>
                                        </table>
                                    </div>

                        <!-- Modal -->
                        <div class="modal fade" id="exampleModal" tabindex="-1" aria-labelledby="exampleModalLabel"
                             aria-hidden="true">
                                      <div class="modal-dialog">
                                        <div class="modal-content">
                                          <div class="modal-header">
                                            <h5 class="modal-title" id="exampleModalLabel">New</h5>
                                              <button type="button" class="btn-close" data-bs-dismiss="modal"
                                                      aria-label="Close"></button>
                                          </div>
                                            <div class="modal-body">
                                            <div class="container">
                                                <input type="text" class="form-control m-2 " placeholder="Name"
                                                       t-model="state.partner.name"/>
                                                <input type="text" class="form-control m-2 " placeholder="Email"
                                                       t-model="state.partner.email"/>
                                                <input type="text" class="form-control m-2 " placeholder="Phone"
                                                       t-model="state.partner.phone"/>
                                            </div>
                                          </div>
                                            <div class="modal-footer">
                                            <button type="button" class="btn btn-secondary m-2" data-bs-dismiss="modal">Close</button>
                                                <button type="button" class="btn btn-primary m-2"
                                                        data-bs-dismiss="modal"
                                                        t-on-click="savePartner">Save</button>
                                          </div>
                                        </div>
                                      </div>
                                    </div>
                                </div>
                  </t>
                    <t t-set-slot="page_2" title="'RPC Service'" isVisible="true">
                        <button class="btn btn-outline-primary"
                                t-on-click="callingRPCService">RPC Services</button>
                  </t>
                    <t t-set-slot="page_3" title="'HTTP Service'" isVisible="true">
                        <button class="m-5 btn btn-outline-primary"
                                t-on-click="getTodoList">HTTP Services</button>
                        <div style="max-height: 500px; overflow-y: auto;">
                                        <table class="table">
                                            <thead class="sticky-top bg-dark text-white">
                                                <tr>
                                                    <th scope="col">#</th>
                                                    <th scope="col">Title</th>
                                                    <th scope="col">Completed</th>
                                                </tr>
                                            </thead>
                                            <tbody>
                                                <t t-foreach="state.todos" t-as="todo" t-key="todo.id">
                                                    <tr>
                                                        <th scope="row"><span t-esc="todo_index+1"/></th>
                                                        <td>
                                                            <t t-if="todo.title">
                                                                <span t-esc="todo.title"/>
                                                            </t>
                                                            <t t-else="">
                                                                <span>-</span>
                                                            </t>
                                                        </td><td>
                                                            <span t-esc="todo.completed"/>
                                                        </td>

                                                    </tr>
                                                </t>
                                            </tbody>
                                        </table>
                                    </div>
                  </t>
                    <t t-set-slot="page_4" title="'Other Services'" isVisible="true">
                        <h1><t t-esc="display.controlPanel.topRight"/></h1>
                        <button class="btn btn-primary m-5" t-on-click="changeRouter">Router Service</button>
                        <button class="btn btn-primary m-5" t-on-click="getCompany">Company Service</button>
                  </t>
                </Notebook>
            </div>
        </Layout>
        </div>

    </t>
</templates>

custom-vf/ica_movie/static/src/ica_movie/IcaMovieActionER.js
/** @odoo-module **/

// new file.js
import {registry} from "@web/core/registry";
import IcaMovieAction from "./ica_movie";
import {patch} from "@web/core/utils/patch";
import { onWillRender,
    onWillStart,
    onRendered,
    onMounted,
    onWillDestroy } from "@odoo/owl";

patch(IcaMovieAction.prototype, {
    setup(){
        super.setup(...arguments);
        onWillStart(()=>{
            console.log("onWillStart")
        });
        onWillRender(()=>{
            console.log("onWillRender")
        });
        onRendered(()=>{
            console.log("onRendered")
        });
        onMounted(()=>{
            console.log("onMounted")
        })
        onWillDestroy(()=>{
            console.log("onWillDestroy")
        })
    },
    async callOrmMethod(partner) {
        onWillDestroy(()=>{
            console.log("onWillDestroy")
        })
        // console.log(partner)
        await this.orm.call(this.resModel, "action_class_from_json", [[partner.id]], {
            name: partner.name,
            email: partner.email,
        });
    },

    async callingRPCService() {
        console.log("click")

        await this.rpcService("/rpc/login", {
            username: "username",
            password: "admin"
        });
    },

    async searchPartners() {
        var result = super.searchPartners(...arguments);
        if (result === null) {
            console.log("Hell Search partners Method Extension.")
        }
    }
})


custom-vf/ica_movie/static/src/list_view/list_view.js
/** @odoo-module */
import {Component, onWillStart, useState} from "@odoo/owl";
import {ScrollableComponent} from "../standalone_app/components/scrollable_component/scrollable_component";
import {CustomerList} from "../customer_list/customer_list";

export class ListViewComponent extends Component {
    static template = "ica_movie.ListViewComponent";
    static props = {};
    static components = {ScrollableComponent, CustomerList}
}

custom-vf/ica_movie/static/src/list_view/list_view.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.ListViewComponent">
<!--         partners="props.partners"-->
<!--        <ScrollableComponent class="'bg-danger'">-->
        <ScrollableComponent>
<!--            <t t-set-slot="title">-->
<!--                <h1>List View</h1>-->
<!--            </t>-->

            <table class="table">
                <thead class="sticky-top bg-dark text-white">
                    <tr>
                        <th scope="col">#</th>
                        <th scope="col">Image</th>
                        <th scope="col">Name</th>
                        <th scope="col">Email</th>
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

custom-vf/ica_movie/static/src/sale_orders/sale_orders.js
/** @odoo-module */
import {Component, onWillStart, useState} from "@odoo/owl";
import {registry} from "@web/core/registry";

export class SaleOrders extends Component {
    static template = "ica_movie.SaleOrders";
    static props = {};

    setup() {
        this.state = useState({
            translateText: "ICA",
        })
    }

    getClick(){
        console.log("heloo");
    }
}

registry.category('ica.movie').add('saleOrderScreen', SaleOrders)

custom-vf/ica_movie/static/src/sale_orders/sale_orders.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.SaleOrders">
        <div class="container mt-5">
            <h1>Hello Sale Orders</h1>
            <div class="container m-5">
                <h1><t  t-esc="state.translateText"/></h1>
                <button t-on-click="getClick">Hello</button>
                <button>Hello</button>
            </div>
        </div>
        <t t-call="ica_movie.so_sub_temp">
            <t t-set="translateText" t-value="state.translateText"/>
        </t>
    </t>

    <t t-name="ica_movie.so_sub_temp">
        <h1>Hello</h1>
        <h1><t  t-esc="translateText"/></h1>
    </t>
</templates>


custom-vf/ica_movie/static/src/standalone_app/components/navbar/navbar.js
/** @odoo-module */
import {Component, onWillStart, useState,useSubEnv} from "@odoo/owl";

export class Navbar extends Component {
    static template = "ica_movie.Navbar";
    static props = {};

    setup(){
        console.log(this.env)
    }

    // switchCustomersScreen(){
    //     console.log("Hello")
    //     console.log(this.props)
    //     // this.props.switchScreen('customerScreen')
    // }
}

custom-vf/ica_movie/static/src/standalone_app/components/navbar/navbar.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.Navbar">
        <nav class="navbar navbar-expand-lg navbar-light bg-light">
              <div class="container-fluid">
                <a class="navbar-brand" href="#" t-on-click="()=>env.switchScreen('customerScreen')">ICA</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
                  <span class="navbar-toggler-icon"></span>
                </button>
                <div class="collapse navbar-collapse" id="navbarSupportedContent">
                  <ul class="navbar-nav me-auto mb-2 mb-lg-0">
                    <li class="nav-item" t-on-click="()=>env.switchScreen('customerScreen')">
<!--                        t-attf-class="btn bg-success text-white mx-2 #{state.view ==='list'?'disabled':''}"-->
                      <a class="nav-link" t-attf-class="#{props.mainScreen ==='customerScreen'? 'active fw-bold':''}" aria-current="page" href="#">Home</a>
                    </li>
                    <li class="nav-item" t-on-click="()=>env.switchScreen('saleOrderScreen')">
                      <a class="nav-link" t-attf-class="#{props.mainScreen ==='saleOrderScreen'? 'active fw-bold':''}" href="#">Sale Orders</a>
                    </li>
                      <li class="nav-item">
                      <a class="nav-link disabled" href="#">
                          <t t-esc="props.mainScreen"/>
                      </a>
                    </li>
                  </ul>
<!--                  <form class="d-flex">-->
<!--                    <input class="form-control me-2" type="search" placeholder="Search" aria-label="Search"/>-->
<!--                    <button class="btn btn-outline-success" type="submit">Search</button>-->
<!--                  </form>-->
                </div>
              </div>
            </nav>

    </t>
</templates>

custom-vf/ica_movie/static/src/standalone_app/components/scrollable_component/scrollable_component.js
/** @odoo-module */
import {Component, onWillStart, useState} from "@odoo/owl";

export class ScrollableComponent extends Component {
    static template = "ica_movie.ScrollableComponent";
    static props = {};
    static defaultProps = {
        class: "m-5 bg-white",
    };

    // setup(){
    //     console.log(this.props)
    // }
}

custom-vf/ica_movie/static/src/standalone_app/components/scrollable_component/scrollable_component.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.ScrollableComponent">
        <t t-slot="title"/>
<!--        class="bg-white"-->
        <div t-att-class="props.class"  style="max-height: 80vh; overflow-y: auto;">
            <t t-slot="default"/>
        </div>
    </t>
</templates>

custom-vf/ica_movie/static/src/standalone_app/todo_app/style.scss
button.btn-warning{
  background-color: #000!important;
    color: white!important;
}

custom-vf/ica_movie/static/src/standalone_app/todo_app/todo_list.js
/** @odoo-module */
import {Component, useState, onWillStart, useSubEnv,useChildSubEnv} from "@odoo/owl";
import {registry} from "@web/core/registry";
import { useAutofocus, useService } from "@web/core/utils/hooks";
import {useStore} from "./todo";


export class TodoList extends Component {
    static template = "ica_movie.TodoList";
    static components = {};
    static props = {};

    setup(){
        this.inputRef = useAutofocus({refName:'title'});
        this.store = useStore()
    }

    addTask(){
        this.store.addTask({'name':this.inputRef.el.value})
    }
}

registry.category('ica.movie').add('todo_list', TodoList)

custom-vf/ica_movie/static/src/standalone_app/todo_app/todo_list.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.TodoList">
        <h1>Todo List</h1>
        <div class="m-3 d-flex justify-content-lg-between">
            <input t-ref="title" type="text" placeholder="Title" class="form-control"/>
            <button t-on-click="addTask" class="btn btn-warning">Add Task</button>
        </div>
        <t t-foreach="env.store.todos" t-as="todo" t-key="todo_index">
            <t t-call="ica_movie.TodoItem">
                <t t-set="todo" t-value="todo"/>
<!--                <t t-set="deleteTask" t-value="this.store.deleteTask"/>-->
            </t>
        </t>
    </t>

    <t t-name="ica_movie.TodoItem">
        <link rel="stylesheet" href="/ica_movie/static/src/standalone_app/todo_app/style.scss"/>
         <div class="card">

                <h1 class="card-title">
                    <div class="d-flex justify-content-lg-between">
                        <div>
                            <t t-out="todo.name"/>
                        </div>
                        <div>
                            <button class="btn btn-warning m-2">Update</button>
                            <button class="btn bg-danger m-2" t-on-click="()=>this.store.deleteTask(todo.id)">Delete</button>
                        </div>
                    </div>
                </h1>
         </div>
    </t>
</templates>


custom-vf/ica_movie/static/src/standalone_app/todo_app/todo.js
/** @odoo-module */
import { reactive,useState, useEnv } from "@odoo/owl";

class Todo {
    nextId = 1;
    todos = [
        {'id': 1, 'name': 'hello'},
        {'id': 2, 'name': 'hello 2'}
    ]

    // UserError(_('Hello'))

    addTask(todo){
        todo['id'] = this.nextId++;
        // console.log(todo);
        this.todos.push(todo);
        // console.log(this.todos.length)
    }

    deleteTask(id){
        // console.log(id)
        this.todos = this.todos.filter((todo) => todo.id !== id)
    }
}

export function createTodoStore() {
    return reactive(new Todo());
}

export function useStore() {
    const env = useEnv();
    return useState(env.store);
}


custom-vf/ica_movie/static/src/standalone_app/app.js
/** @odoo-module */
import {whenReady} from "@odoo/owl";
import {mountComponent} from "@web/env";
import {Root} from "./root";
// import {createTodoStore} from "./todo_app/todo";

whenReady(() => mountComponent(Root, document.body));

custom-vf/ica_movie/static/src/standalone_app/root.js
/** @odoo-module */
import {Component, useState, onWillStart, useSubEnv, useChildSubEnv, useEnv} from "@odoo/owl";
import {Navbar} from "./components/navbar/navbar";
import {registry} from "@web/core/registry";
import {createTodoStore, useStore} from "./todo_app/todo";


export class Root extends Component {
    static template = "ica_movie.Root";
    static components = {Navbar};
    static props = {};

    setup() {
        this.state = useState({
            mainScreen: 'todo_list',
        })
        // <= 30
        useSubEnv({
            switchScreen: this.switchScreen.bind(this),
            store: createTodoStore()
        })
    }

    switchScreen(name) {
        // console.log("Hello I am from root  of switch screen")
        // console.log(name)
        this.state.mainScreen = name;
    }

    getComponent() {
        // return Customers;
        let mainScreen = this.state.mainScreen
        return registry.category('ica.movie').get(mainScreen);
        // return this.state.mainScreen === 'saleOrderScreen' ? Customers : SaleOrders;
    }
}

custom-vf/ica_movie/static/src/standalone_app/root.xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="ica_movie.Root">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js" integrity="sha384-MrcW6ZMFYlzcLA8Nl+NtUVF0sA7MsXsP1UyJoMp4YLEuNSfAP+JcXn/tWtIaxVXM" crossorigin="anonymous"></script>
<!--        <Navbar/>-->

        <Navbar switchScreen.bind="switchScreen" mainScreen="state.mainScreen"/>
        <t t-call-assets="ica_movie.custom_assets"/>

<!--        <div class="container mt-5">-->
<!--            <div class="buttons">-->
<!--              <button class="button is-primary">Primary</button>-->
<!--                <button class="button is-link">Link</button>-->
<!--            </div>-->

<!--            <div class="buttons">-->
<!--              <button class="button is-info">Info</button>-->
<!--                <button class="button is-success">Success</button>-->
<!--                <button class="button is-warning">Warning</button>-->
<!--                <button class="button is-danger">Danger</button>-->
<!--            </div>-->
<!--        </div>-->


        <!--        <div class="container">-->
        <!--            <button t-on-click="()=>this.switchScreen('customerScreen')" class="btn btn-primary">Customer</button>-->
        <!--            <button t-on-click="()=>this.switchScreen('saleOrderScreen')" class="btn btn-primary">SaleOrders</button>-->
        <!--        </div>-->

        <t t-component="getComponent()"/>
        <!--        <Customers/>-->
        <!--        <SaleOrders/>-->
    </t>
</templates>

custom-vf/ica_movie/views/ica_movie_client_action.xml
<?xml version="1.0" encoding="UTF-8" ?>
<odoo>
    <record model="ir.actions.client" id="ica_movie_action">
        <field name="name">Movies</field>
        <field name="tag">ica_movie.movieAction</field>
    </record>

    <record id="ica_standalone_action" model="ir.actions.act_url">
        <field name="name">Standalone</field>
        <field name="url">/ica-movie/standalone_app</field>
    </record>

<!--     This Menu Item will appear in the Upper bar, That's why It needs NO parent or action -->
    <menuitem id="ica_movie_root" name="ICA">
        <!-- This Menu Item must have a parent and an action -->
        <menuitem id="movie_category" name="Movie" action="ica_movie_action" sequence="0"/>
        <menuitem id="standalone_category" name="Standalone" action="ica_standalone_action" sequence="1"/>
    </menuitem>
</odoo>

custom-vf/ica_movie/views/template.xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <template id="ica_movie.standalone_app">
<!--            <t t-call="web.layout">-->
                &lt;!DOCTYPE html&gt;
            <html>
                <head>
                    <script type="text/javascript">
                        var odoo = {
                        csrf_token: "<t t-nocache="The csrf token must always be up to date."
                                        t-esc="request.csrf_token(None)"/>",
                        debug: "<t t-out="debug"/>",
                        __session_info__: <t t-esc="json.dumps(session_info)"/>,
                        };
                    </script>
                    <t t-call-assets="ica_movie.assets_standalone_app"/>
                </head>
                <body/>
            </html>
<!--            </t>-->
    </template>
</odoo>

custom-vf/ica_movie/__manifest__.py
{
    "name": "ICA  Movie",
    "depends": ["base", "web", "mail"],
    "license": "LGPL-3",
    "data": [
        "views/ica_movie_client_action.xml",
        "views/template.xml",
    ],
    "assets": {
        "web.assets_backend": [
            "/ica_movie/static/src/ica_movie/*",
        ],
        'ica_movie.assets_standalone_app': [
            ('include', 'web._assets_helpers'),
            'web/static/src/scss/pre_variables.scss',
            'web/static/lib/bootstrap/scss/_variables.scss',
            ('include', 'web._assets_bootstrap'),
            ('include', 'web._assets_core'),
            'web/static/src/libs/fontawesome/css/font-awesome.css',
            'web/static/lib/odoo_ui_icons/*',
            'ica_movie/static/src/standalone_app/**/*.js',
            'ica_movie/static/src/standalone_app/**/*.xml',
            'ica_movie/static/src/standalone_app/**/*.scss',
        ],
        # "ica_movie.custom_assets": [
        #     'ica_movie/static/src/standalone_app/**/*.scss',
        # ],
    }
}

"""