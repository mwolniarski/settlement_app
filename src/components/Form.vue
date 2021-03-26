<template>
  <div class="container bg-light">
    <div v-if="!recordUpdated">
      <Alert v-bind:isDisplayed="totalAmountFromDocuments<paymentAmount" />
      <div class="form-row p-2">
          <div class="col">
              <label for="date">Wybierz datę zapłaty</label>
          </div>
          <div class="col">
              <input type="date" v-model="paymentDate" class="form-control" id="date" />
          </div>
      </div>  
      <div class="form-row p-2">
          <div class="col">
              <label for="comments">Uwagi</label>
          </div>
          <div class="col">
              <textarea name="comments" v-model="comments" rows="5" class="form-control"></textarea>
          </div>
      </div>
      <div class="form-row p-2">
          <div class="form-group col">
              <label for="method">Metoda płatności</label>
          </div>
          <div class="form-group col">
              <select class="form-control" id="method" v-model="selectedPaymentMethod">
                  <option v-for="paymentMethod in paymentMethods" v-bind:key='paymentMethod.text' v-bind:value='paymentMethod.text'>{{paymentMethod.text}}</option>
              </select>
          </div>
      </div>
      <div class="form-row p-2">
          <div class="col">
              <label for="amount">Kwota wpłaty</label>
          </div>
          <div class="col">
              <input type="number" v-model="paymentAmount" class="form-control" id="amount"/>
          </div>
      </div>            
      <FormButtons @close="close" @settlementDocuments="settlementDocuments"/>
    </div>
    <EndPanel v-if="recordUpdated" v-bind:endMessage="endMessage" @close="close"/>
  </div>
</template>

<script>
import Alert from './formComponents/Alert'
import FormButtons from './formComponents/FormButtons'
import EndPanel from './EndPanel'

const ZOHO = window.ZOHO;

export default {
  name: 'Form',
  components:{
    Alert,
    FormButtons,
    EndPanel
  },
  data(){
    return{
      paymentDate: '',
      comments: '',
      selectedPaymentMethod: "Przelew",
      paymentAmount: 0,
      totalAmountFromDocuments: 0,
      recordList: [],
      paymentMethods:[
        {text:"Przelew"},
        {text:"Gotówka"},
        {text:"Przedpłata"},
        {text:"Kompensata"}
      ],
      recordUpdated: false,
      endMessage: "Żaden z wybranych dokumentów nie został całkowicie opłacony"
    }
  },
  methods:{
    settlementDocuments(){
      this.recordList.sort(this.compare);
      this.recordList.forEach(element => this.updateDocuments(element));
      this.recordUpdated = true;
    },
    close(){
      ZOHO.CRM.UI.Popup.close();
    },
    async onEmbeddedApp(data){
        data['EntityId'].forEach(async element => this.getRecord(element));
    },
    async getRecord(id){
        let response = await ZOHO.CRM.API.getRecord({Entity:"Dokumenty_ksi_gowe",RecordID:id});
        let record = response['data'][0];
        this.recordList.push(record)
        if(record['Pozosta_o_do_zap_aty'] != null)
          this.totalAmountFromDocuments += record['Pozosta_o_do_zap_aty']
        else
          this.totalAmountFromDocuments += record['Razem_brutto']
        this.paymentAmount = this.totalAmountFromDocuments
    },
    async updateDocuments(element){
      let amount;
      if(element['Pozosta_o_do_zap_aty'] != null)
        amount = element['Pozosta_o_do_zap_aty'];
      else
        amount = element['Razem_brutto'];
      
      if(this.paymentAmount < amount){
        amount = this.paymentAmount
        this.paymentAmount = 0;
      }
      else{
        if(this.endMessage == "Żaden z wybranych dokumentów nie został całkowicie opłacony")
          this.endMessage = "Te dokumenty zostały całkowicie opłacone: <br/>"
        this.paymentAmount -= amount;
        this.endMessage += (element['Name'] + "<br/>")
      }
      if(amount != 0){
        let apiData = new Map();
        let paymentNumber, paymentMehodFieldName;
        if(element["Kwota_przelewu_cz_ciowego_1"] == null)
        {
          paymentNumber = 1;
          paymentMehodFieldName = "Spos_b_zap_aty";
        }
        else if(element["Kwota_przelewu_cz_ciowego_2"] == null)
        {
          paymentNumber = 2;
          paymentMehodFieldName = "Spos_b_zap_aty_nr_2";
        }
        else if(element["Kwota_przelewu_cz_ciowego_3"] == null)
        {
          paymentNumber = 3;
          paymentMehodFieldName = "Spos_b_zap_aty_nr_3";
        }
        else if(element["Kwota_przelewu_cz_ciowego_4"] == null)
        {
          paymentNumber = 4;
          paymentMehodFieldName = "Spos_b_zap_aty_nr_4";
        }
        else if(element["Kwota_przelewu_cz_ciowego_5"] == null)
        {
          paymentNumber = 5;
          paymentMehodFieldName = "Spos_b_zap_aty_nr_5";
        }
        
        let fieldName = "Kwota_przelewu_cz_ciowego_"+paymentNumber;
        let checkbox = "Przelew_cz_ciowy_nr_"+paymentNumber;
        let dateFieldName = "Data_zap_aty_nr_"+paymentNumber;
        let commentFieldName = "Szczeg_y_przelewu_cz_ciowego_"+paymentNumber;

        apiData.set(fieldName, amount);
        apiData.set(checkbox, true);
        apiData.set('id', element['id']);
        apiData.set(dateFieldName, this.paymentDate);
        apiData.set(commentFieldName, this.comments);	
        apiData.set(paymentMehodFieldName, this.selectedPaymentMethod);
        let config = {
          Entity:"Dokumenty_ksi_gowe",
          APIData:Object.fromEntries(apiData),
          Trigger:["workflow"]
        }
        await ZOHO.CRM.API.updateRecord(config);
      }
    },
    compare(a, b) {
      if (a['Termin_platnosci_w_dniach'] < b['Termin_platnosci_w_dniach'])
          return -1
      if (a['Termin_platnosci_w_dniach'] > b['Termin_platnosci_w_dniach'])
          return 1
      return 0
    }
  },
  async created(){
    await ZOHO.embeddedApp.on("PageLoad", (data) => this.onEmbeddedApp(data));
    await ZOHO.embeddedApp.init();

    let dateTmp = new Date(Date.now());
    let day = dateTmp.getDate();
    let month = dateTmp.getMonth()+1;
    let year = dateTmp.getFullYear();
    if(day < 10)
      day = "0" + day;
    if(month < 10)
      month = "0" + month;
    this.paymentDate = year + "-" + month + "-" + day;
  }
}
</script>