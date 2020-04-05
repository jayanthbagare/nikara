
<template>
  <div class="container">
    <b-alert
      style="z-index:2000;"
      v-model="successAlert"
      variant="success"
      dismissible
    >{{successMessage}}</b-alert>

    <h2>Sharmalaya - Events</h2>
    <br>
    <br>
    <b-form-select v-model="optionSelected" :options="options" v-on:change="processSelectedEvent"></b-form-select>

    <template v-if="event.id > 0">
      <br>
      <br>
      <br>
      <b-card>
        <b-card-text>
          <strong>Event:</strong>
          {{ event.title }}
        </b-card-text>
        <b-card-text>
          <strong>Details:</strong>
          {{ event.description }}
        </b-card-text>
        <b-card-text>
          <strong>Start:</strong>
          {{ event.startdatetime }}
        </b-card-text>
        <b-card-text>
          <strong>End:</strong>
          {{ event.enddatetime }}
        </b-card-text>
        <b-card-text>
          <strong>Cost Per Child:</strong>
          {{ event.cost_child }}
        </b-card-text>
        <b-card-text>
          <strong>Cost Per Adult:</strong>
          {{ event.cost_adult }}
        </b-card-text>
      </b-card>
    </template>
    <br>
    <br>
    <br>
    <br>
    <div></div>
    <div>
      <div>
        <template v-if="event.id > 0">
          <b-badge variant="outline-info" class="float-left">Choose Names & Register</b-badge>
          <b-badge
            variant="outline-info"
            class="float-right"
          >You have to pay: &#x20b9; {{ totalCost }}</b-badge>
        </template>
        <template v-else>
          <b-badge variant="secondary">Your Members</b-badge>
        </template>
        <b-table
          ref="memberTable"
          selectable
          :fields="fields"
          @row-selected="onRowSelected"
          responsive
          hover
          :items="memberList"
          primary-key="id"
        >
          <template v-slot:cell(Gender)="data">
            <template v-if="data.Gender=1">
              <span>Female</span>
            </template>
            <template v-else>
              <span>Male</span>
            </template>
          </template>

          <template v-slot:cell(selected)="{ rowSelected }">
            <template v-if="rowSelected">
              <span aria-hidden="true">&check;</span>
              <span class="sr-only">Selected</span>
            </template>
            <template v-else>
              <span aria-hidden="true">&nbsp;</span>
              <span class="sr-only">Not selected</span>
            </template>
          </template>
        </b-table>
      </div>
    </div>
    <template v-if="event.id > 0">
      <b-button
        v-on:click="registerMembers"
        variant="outline-secondary"
        size="sm"
      >Register Selected Members</b-button>
    </template>
    <b-button
      v-b-modal.addMemberModal
      variant="outline-secondary"
      class="float-right"
      size="sm"
    >Add a new Member</b-button>
    <b-modal
      id="addMemberModal"
      centered
      ok-variant="outline-secondary"
      ok-only
      ok-title="Close Form"
    >
      <template>
        <div>
          <b-form v-if="show">
            <b-form-group id="input-group1" label="Name" label-for="input-1">
              <b-form-input
                id="input-1"
                v-model="memberForm.name"
                type="text"
                required
                placeholder="Enter your Full Name"
              ></b-form-input>
            </b-form-group>

            <b-form-group id="input-group2" label="Date of Birth" label-for="input-2">
              <b-form-datepicker id="input-2" v-model="memberForm.dob" variant="secondary" required></b-form-datepicker>
            </b-form-group>

            <b-form-group label="Gender">
              <b-form-radio v-model="memberForm.gender" value="1">Female</b-form-radio>
              <b-form-radio v-model="memberForm.gender" value="2">Male</b-form-radio>
            </b-form-group>
          </b-form>
          <b-form-group id="input-group4" label="Mobile" label-for="input-4">
            <b-form-input
              id="input-4"
              v-model="memberForm.phone"
              type="number"
              placeholder="Enter your Mobile Number"
            ></b-form-input>
          </b-form-group>
          <b-button
            type="submit"
            class="float-right"
            variant="outline-secondary"
            v-on:click="onSubmit"
          >Submit</b-button>
        </div>
      </template>
    </b-modal>
  </div>
</template>

<script>
import axios from "axios";
import moment from "moment";

axios.defaults.xsrfHeaderName = "X-CSRFTOKEN";
axios.defaults.xsrfCookieName = "csrftoken";
axios.defaults.withCredentials = true;

export default {
  asyncData() {
    console.log(process.env);
    var graphql_url = process.env.HASURA_URL;
    var eventList = [];
    var event = {};
    var memberList = [];
    var selected = [];
    var regMembers = [];
    var successAlert = false;
    var authorId = 1;
    var show = true;
    var successMessage = "";
    var memberForm = {
      name: "",
      email: "",
      gender: "",
      phone: "",
      dob: ""
    };
    var HTTPheaders = {
      "Content-Type": "application/json",
      Accept: "application/json",
      "X-Hasura-User-Id": authorId,
      "X-Hasura-Role": process.env.X_Hasura_Role,
      "x-hasura-admin-secret": process.env.X_Hasura_Admin_Secret
    };
    var memberFields = [
      //{ key: "id", label: "id" },
      { key: "Selected", label: "Reg" },
      { key: "Name", label: "Name" },
      { key: "Gender", label: "Gender" },
      { key: "Age", label: "Age" },
      { key: "Phone", label: "Phone" }
    ];

    var optionSelected = null;
    var options = [{ value: null, text: "Choose an Event to Register to" }];
    var totalCost = 0;

    return {
      graphql_url: graphql_url,
      eventList: eventList,
      memberList: memberList,
      selected: selected,
      fields: memberFields,
      successAlert: successAlert,
      optionSelected: optionSelected,
      options: options,
      event: event,
      totalCost: totalCost,
      authorId: authorId,
      HTTPheaders: HTTPheaders,
      regMembers: regMembers,
      memberForm: memberForm,
      show: show,
      successMessage: successMessage
    };
  },
  async created() {
    this.getAllEvents();
    this.getAllMyMembers();
  },
  methods: {
    onSubmit: async function(evt) {
      evt.preventDefault();
      if (this.memberForm.phone === "") {
        this.memberForm.phone = null;
      }

      await axios({
        method: "POST",
        url: this.graphql_url,
        data: {
          query: `
              mutation upsert_members($name:String!,$gender:Int!,$dob:date!,$phone:numeric,$author:Int!) {
                insert_Members(objects: [{
                              Name:$name,
                              Gender:$gender,
                              Date_Of_Birth:$dob,
                              Phone:$phone,
                              author:$author
                              }])
                              {
                                affected_rows
                              }
                            }`,
          variables: {
            name: this.memberForm.name,
            gender: this.memberForm.gender,
            dob: this.memberForm.dob,
            phone: this.memberForm.phone,
            author: this.authorId
          }
        },
        headers: this.HTTPheaders
      })
        .then(res => {
          //make an object and update the member list
          var tmp_member = {
            Name: this.memberForm.name,
            Gender: this.memberForm.gender,
            Date_Of_Birth: this.memberForm.dob,
            Phone: this.memberForm.phone,
            Age: this.calculateAge(this.memberForm.dob)
          };
          this.memberList.push(tmp_member);
          tmp_member = undefined;

          //Clear the form
          this.memberForm.name = undefined;
          this.memberForm.gender = undefined;
          this.memberForm.dob = undefined;
          this.memberForm.phone = undefined;

          //Hide the modal
          this.$bvModal.hide("addMemberModal");
          // show the alert for success message.
          this.successMessage = "Saved the Member successfully!!";
          this.successAlert = true;
          //console.info(res);
        })
        .catch(err => console.error(err));
    },
    calculateAge: function(date_of_birth) {
      var age = moment().diff(date_of_birth, "years");
      return age;
    },
    calculateTotalCost: function(selectedMemberArray) {
      this.totalCost = 0;
      var age = 0;
      for (var j = 0; j < selectedMemberArray.length; j++) {
        age = moment().diff(selectedMemberArray[j].Date_Of_Birth, "years");

        if (age < 12) {
          this.totalCost += this.event.cost_child;
        } else if (age == NaN) {
          this.totalCost += this.event.cost_adult;
        } else {
          this.totalCost += this.event.cost_adult;
        }
      }
    },
    getAllMyMembers: async function() {
      try {
        var members = await axios({
          method: "POST",
          url: this.graphql_url,
          data: {
            query: `
                  {
                  Members
                    {
                      id
                      Name
                      Gender
                      Date_Of_Birth
                      Phone
                    }
                  }
                `
          },
          headers: this.HTTPheaders
        });
      } catch (err) {
        console.error(err);
      }
      this.memberList = members.data.data.Members;

      for (var z = 0; z < this.memberList.length; z++) {
        console.log("Inside for");
        this.memberList[z].Age = moment().diff(
          this.memberList[z].Date_Of_Birth,
          "years"
        );
      }
      console.log(this.memberList);
    },
    getAllEvents: async function() {
      try {
        var result = await axios({
          method: "POST",
          url: this.graphql_url,
          data: {
            query: `
                  {
                  Event
                    {
                        id
                        title
                        startdatetime
                    }
                  }
                `
          },
          headers: this.HTTPheaders
        });
        //this.people = result.data.data.people;
      } catch (error) {
        console.error(error);
      }

      this.eventList = result.data.data.Event;

      for (var m = 0; m < this.eventList.length; m++) {
        var optionsText =
          this.eventList[m].title +
          "-" +
          moment(this.eventList[m].startDatetime).format(
            "ddd, MMM Do YYYY, h:mm:ss a"
          );

        var ops = {
          value: this.eventList[m].id,
          text: optionsText
        };
        this.options.push(ops);
      }
    },
    returnCurrentEventId: function() {
      return this.event.id;
    },
    processSelectedEvent: async function(args) {
      if (args != null) {
        this.registeredMembers = [];
        this.$refs.memberTable.clearSelected();
        var res = await axios({
          method: "POST",
          url: this.graphql_url,
          data: {
            query: `
              query getSingleEvent($event:Int!){
                Event_by_pk(id:$event){
                    id
                    title
                    description
                    remarks
                    startdatetime
                    enddatetime
                    cost_adult
                    cost_child
                    remarks
                }
            }`,
            variables: {
              event: args
            }
          },
          headers: this.HTTPheaders
        })
          .then(res => {
            this.event = res.data.data.Event_by_pk;
            console.info(res);

            this.event.startdatetime = moment(this.event.startdatetime).format(
              "ddd, MMM Do YYYY, h:mm:ss a"
            );
            if (this.event.enddatetime != null) {
              this.event.enddatetime = moment(this.event.enddatetime).format(
                "ddd, MMM Do YYYY, h:mm:ss a"
              );
            }
          })
          .catch(err => console.error(err));

        var registeredMembers = await axios({
          method: "POST",
          url: this.graphql_url,
          data: {
            query: `
              query getRegisteredMembers($event:Int!,$author:Int!) {
                  Event_Member(
                    where: {
                      _and: 
                        [
                          {EventId:{_eq:$event}},
                          {AuthorId:{_eq:$author}}
                        ], 
                      }
                  ) {
                      MemberId
                    }
                  }`,
            variables: {
              event: args,
              author: this.authorId
            }
          },
          headers: this.HTTPheaders
        })
          .then(res => {
            this.regMembers = res.data.data.Event_Member;
            for (var m = 0; m < this.regMembers.length; m++) {
              for (var n = 0; n < this.$refs.memberTable.items.length; n++) {
                if (
                  this.$refs.memberTable.items[n].id ==
                  this.regMembers[m].MemberId
                ) {
                  this.$refs.memberTable.selectRow(n);
                }
              }
              this.selected.push(this.regMembers[m].MemberId);
            }

            console.info(res.data.data.Event_Member);
          })
          .catch(err => console.error(err));
      }
    },
    onRowSelected(items) {
      this.calculateTotalCost(items);
      this.selected = items;
    },

    registerMembers: async function(event) {
      var chosen = "";
      var del_rows = await axios({
        method: "POST",
        url: this.graphql_url,
        data: {
          query: `
            mutation delete_event_member($event: Int!, $author: Int!) {
                  delete_Event_Member(where: {_and: [{EventId: {_eq: $event}}, {AuthorId: {_eq: $author}}]}){
                  affected_rows
                }
              }`,
          variables: {
            event: this.event.id,
            author: this.authorId
          }
        },
        headers: this.HTTPheaders
      })
        .then(res => {
          console.info(res);
        })
        .catch(err => {
          console.error(err);
        });
      for (var i = 0; i < this.selected.length; i++) {
        await axios({
          method: "POST",
          url: this.graphql_url,
          data: {
            query: `
              mutation insertEventMember($event:Int!,$member:Int!,$author:Int!){
                insert_Event_Member(objects:{EventId:$event,MemberId:$member,AuthorId:$author}){
                  affected_rows
                }
            }`,
            variables: {
              member: this.selected[i].id,
              event: this.event.id,
              author: this.authorId
            }
          },
          headers: this.HTTPheaders
        })
          .then(res => {
            console.info(res);
          })
          .catch(err => console.error(err));
      }
      this.successMessage = "Registered the Members Successfully!!";
      this.successAlert = true;
    }
  }
};
</script>

<style>
</style>