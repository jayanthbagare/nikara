
<template>
  <div class="container">
    <h2>Sharmalaya - Events</h2>
    <br />
    <br />
    <b-form-select v-model="optionSelected" :options="options" v-on:change="processSelectedEvent"></b-form-select>

    <template v-if="event.id > 0">
      <br />
      <br />
      <br />
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
    <br />
    <br />
    <br />
    <br />
    <div>
      <b-alert
        v-model="successAlert"
        variant="success"
        dismissible
      >Sucessfully Registered Selected Memers to the Event!</b-alert>
    </div>
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
      ok-only
      ok-variant="outline-secondary"
      ok-title="Add Member"
    >
      <template>
        <b-container fluid>
          <b-row>
            <b-col sm="4">
              <label for="txtName">Name</label>
            </b-col>
            <b-col sm="8">
              <b-form-input type="text" id="txtName"></b-form-input>
            </b-col>
          </b-row>
          <br />
          <b-row>
            <b-col sm="4">
              <label for="txtPhonne">Phone</label>
            </b-col>
            <b-col sm="8">
              <b-form-input type="tel" id="txtPhone"></b-form-input>
            </b-col>
          </b-row>
          <br />
          <b-row>
            <b-col sm="4">
              <b-form-group label="Gender">
                <b-col sm="8">
                  <b-form-radio v-model="selected" name="radGender" value="F">Female</b-form-radio>
                  <b-form-radio v-model="selected" name="radGender" value="M">Male</b-form-radio>
                </b-col>
              </b-form-group>
            </b-col>
          </b-row>
          <br />
          <b-row>
            <b-col sm="4">
              <label for="datDOB">Date of Birth</label>
            </b-col>
            <b-col sm="8">
              <b-form-input type="date" id="datDOB"></b-form-input>
            </b-col>
          </b-row>
        </b-container>
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
    var graphql_url = process.env.HASURA_URL;
    var eventList = [];
    var event = {};
    var memberList = [];
    var selected = [];
    var regMembers = [];
    var successAlert = false;
    var authorId = 2;
    var HTTPheaders = {
      "Content-Type": "application/json",
      "X-Hasura-User-Id": authorId,
      "X-Hasura-Role": process.env.X-Hasura-Role,
      "X-Hasura-Admin-Secret": process.env.X-Hasura-Admin-Secret,
    };
    var memberFields = [
      {key:"id",label:"id"},
      { key: "Selected", label: "Registered" },
      { key: "Name", label: "Full Name" },
      { key: "Gender", label: "Gender" },
      { key: "Date_Of_Birth", label: "Date of Birth" },
      { key: "Phone", label: "Phone" }
    ];
    var optionSelected = null;
    var options = [{ value: null, text: "Choose an Event to Register to" }];
    var totalCost = 0;

    return {
      graphql_url:graphql_url,
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
      regMembers: regMembers
    };
  },
  async created() {
    this.getAllEvents();
    this.getAllMyMembers();
  },
  methods: {
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
        console.error(error);
      }
      console.log(members);

      this.memberList = members.data.data.Members;
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
              for(var n = 0; n< this.$refs.memberTable.items.length; n++) {
                if(this.$refs.memberTable.items[n].id == this.regMembers[m].MemberId){
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
      for (var i = 0; i < this.selected.length; i++) {
        //First delete the existing record, then insert a new one

        var del_rows = await axios({
          method:"POST",
          url:this.graphql_url,
          data:{
            query: `
            mutation delete_event_member($event: Int!, $member: Int!, $author: Int!) {
                  delete_Event_Member(where: {_and: [{EventId: {_eq: $event}}, {MemberId: {_eq: $member}}, {AuthorId: {_eq: $author}}]}){
                  affected_rows
                }
              }`,
              variables: {
                event:this.event.id,
                member:this.selected[i].id,
                author:this.authorId
              }
          },
          headers: this.HTTPheaders
        })
        .then(res => {
          console.info(res)
        })
        .catch(err => {
          console.error(err)
        });

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
      
      this.successAlert = true;
    }
  }
};
</script>

<style>
</style>