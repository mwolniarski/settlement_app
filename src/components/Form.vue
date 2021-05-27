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
        <div class="form-check">
          <input class="form-check-input" type="checkbox" v-model="showExchangeRate" id="showExchange" :disabled="(this.recordList.length+this.dkRecordsList.length)>1">
          <label class="form-check-label" for="showExchange">
            Wpłata w innej walucie
          </label>
        </div>
      </div>
      <div v-if="showExchangeRate">
        <div class="form-row p-2">
          <div class="form-group col">
            <label for="currency">Waluta</label>
          </div>
          <div class="form-group col">
            <select class="form-control" id="currency" v-model="curr">
              <option>EUR</option>
              <option>PLN</option>
            </select>
          </div>
        </div>

        <div class="form-row p-2">
          <div class="form-group col">
            <label for="calculatedAmount">Kwota w wybranej walucie</label>
          </div>
          <div class="form-group col">
            <input class="form-control" type="number" id="calculatedAmount" v-model="amountInOtherCurr">
          </div>
        </div>

        <div class="form-row p-2">
          <div class="form-group col">
            <label for="exchangeDate">Kurs dla przeliczania EUR</label>
          </div>
          <div class="form-group col">
            <input class="form-control" type="date" id="exchangeDate" v-model="exchangeRateDate" disabled>
          </div>
          <div class="form-group col">
            <input class="form-control" type="number" id="exchangeRate" v-model="exchangeRate" disabled>
          </div>
        </div>

      </div>
      <div class="form-row p-2">
          <div class="col">
              <label for="amount">Kwota wpłaty</label>
          </div>
          <div class="col">
              <input type="number" v-model="paymentAmount" class="form-control" id="amount"/>
          </div>
          [{{documentsCurr}}]
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
      curr: '',
      documentsCurr: '',
      amountInOtherCurr: '',
      exchangeRateDate: '',
      exchangeRate: '',
      paymentDate: '',
      comments: '',
      selectedPaymentMethod: "Przelew",
      paymentAmount: 0.0,
      totalAmountFromDocuments: 0.0,
      totalAmountFromDK: 0.0,
      totalAmountFromFV: 0.0,
      recordList: [],
      dkRecordsList: [],
      showExchangeRate: false,
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
    cut(number){
      number = number.toString();
      let index = number.indexOf('.');
      let newNumber = number.substring(0, index+3);
      return newNumber;
    },
    updateTotalAmount(){
      if(this.curr != this.documentsCurr){
        if(this.curr == "EUR"){
          this.paymentAmount = parseFloat(this.amountInOtherCurr*this.exchangeRate).toFixed(2);
        }
        else{
          this.paymentAmount = parseFloat(this.amountInOtherCurr/this.exchangeRate).toFixed(2);
        }
      }
      else{
         this.paymentAmount = parseFloat(this.amountInOtherCurr).toFixed(2);
      }
    },
    settlementDocuments(){
      this.recordList.sort(this.compare);
      this.dkRecordsList.sort(this.compare);

      // this.paymentAmount = this.paymentAmount.toFixed(2);

      console.log("Amount - " + this.paymentAmount)
      

      console.log("DK - " + this.totalAmountFromDK)
      console.log("Amount2 - " + this.paymentAmount)
      console.log("DK2 - " + (this.totalAmountFromDK*2))

      if(this.totalAmountFromDK < this.totalAmountFromFV){
        this.paymentAmount =  parseFloat(this.paymentAmount) + (this.totalAmountFromDK*2);
        console.log(this.paymentAmount);
        this.dkRecordsList.forEach(element => this.updateDocuments(element));
        this.recordList.forEach(element => this.updateDocuments(element));
      }
      else{
        this.paymentAmount = parseFloat(this.paymentAmount) + (this.totalAmountFromFV + this.totalAmountFromDK);
        console.log(this.paymentAmount);
        this.recordList.forEach(element => this.updateDocuments(element));
        this.dkRecordsList.forEach(element => this.updateDocuments(element));
      }
      
      
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
        if(this.documentsCurr != record["Currency"] && this.documentsCurr != ""){
          this.endMessage = "Dokumenty są w różnych walutach!!!";
          this.recordUpdated = true;
        }
        this.documentsCurr = record["Currency"];
        console.log("-------Kwoty--------")
        console.log(record['Pozosta_o_do_zap_aty'])
        console.log(record['Razem_brutto'])
        console.log("--------------------")
        if(record['Layout']['id'] == "208100000001272734"){
          if(record['Pozosta_o_do_zap_aty'] != null){
            this.totalAmountFromDocuments += record['Pozosta_o_do_zap_aty'];
            this.totalAmountFromFV += record['Pozosta_o_do_zap_aty'];
          }
          else{
            this.totalAmountFromDocuments += record['Razem_brutto'];
            this.totalAmountFromFV += record['Razem_brutto'];
          }
          this.recordList.push(record);
        }
        else if(record['Layout']['id'] == "208100000003064001"){
          if(record['Pozosta_o_do_zap_aty'] != null){
            this.totalAmountFromDocuments -= record['Pozosta_o_do_zap_aty'];
            this.totalAmountFromDK += record['Pozosta_o_do_zap_aty'];
          }
          else{
            this.totalAmountFromDocuments -= record['Razem_brutto'];
            this.totalAmountFromDK += record['Razem_brutto'];
          }
          this.dkRecordsList.push(record);
        }
        console.log("-------Total--------")
        console.log(this.paymentAmount)
        console.log(this.totalAmountFromDocuments)
        console.log("--------------------")
        this.paymentAmount = this.totalAmountFromDocuments.toFixed(2);
    },
    async updateDocuments(element){
      if(this.paymentAmount > 0){
        let amount;
        if(element['Pozosta_o_do_zap_aty'] != null)
          amount = element['Pozosta_o_do_zap_aty'];
        else
          amount = element['Razem_brutto'];
        console.log("------------------")
        console.log("Test")
        console.log("------------------")
        console.log(this.paymentAmount)
        console.log(amount)
        console.log("------------------")
        
        if(this.paymentAmount < amount){
          console.log("Nie wystarczyło")
            amount = parseFloat(this.paymentAmount).toFixed(2);
            this.paymentAmount = 0;
        }
        else{
          console.log("Wystarczyło")
          if(this.endMessage == "Żaden z wybranych dokumentów nie został całkowicie opłacony")
            this.endMessage = "Te dokumenty zostały całkowicie opłacone: <br/>";
          this.paymentAmount = parseFloat(this.paymentAmount).toFixed(2) - amount;
          this.endMessage += (element['Name'] + "<br/>");
        }
        console.log(this.paymentAmount)
        console.log(amount)
        if(amount != 0){
          console.log("Rozliczanie")
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
          let fullComment;
          if(this.curr != this.documentsCurr) 
            fullComment = "Wpłata w innej walucie: " + this.amountInOtherCurr + " " + this.curr  + ". Uwagi: " + (this.comments === "" ? "brak": this.comments);
          else
            fullComment = "Uwagi: " + (this.comments === "" ? "brak": this.comments);
          apiData.set(fieldName, amount);
          apiData.set(checkbox, true);
          apiData.set('id', element['id']);
          apiData.set(dateFieldName, this.paymentDate);
          apiData.set(commentFieldName, fullComment);	
          apiData.set(paymentMehodFieldName, this.selectedPaymentMethod);
          let config = {
            Entity:"Dokumenty_ksi_gowe",
            APIData:Object.fromEntries(apiData),
            Trigger:["workflow"]
          }
          console.log("Update")
          let test = await ZOHO.CRM.API.updateRecord(config);
          console.log(test)
        }
      }
    },
    compare(a, b) {
      if (a['Data_platnosci'] < b['Data_platnosci'])
          return -1
      if (a['Data_platnosci'] > b['Data_platnosci'])
          return 1
      return 0
    }
  },
  watch: {
    showExchangeRate(){
      if(this.showExchangeRate){
        let record = this.recordList[0];
        this.exchangeRateDate = record["Tabela_z_dnia"];
        this.exchangeRate = record["Kurs_EUR"];
        this.curr = this.documentsCurr=="EUR" ? "EUR":"PLN"
      }
    },
    amountInOtherCurr(){
      this.updateTotalAmount();
    },
    curr(){
      this.updateTotalAmount();
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