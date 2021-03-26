<template>
  <div class="container bg-light">
    <div v-if="totalAmountFromDocuments<paymentAmount" class="alert alert-danger" role="alert">
 Uwaga nadpłata
</div>
        <div class="form-row p-2">
          <div class="col">
                      <label for="date" >Wybierz datę zapłaty</label>
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
                            <input type="number" v-model="paymentAmount" class="form-control" id="amount" />
                        </div>
                    </div>
                    <div>
                        <button class="btn btn-primary mb-2 mr-2" v-on:click="settlementDocuments()">OK</button>
                        <button class="btn btn-danger mb-2" v-on:click="close()">Anuluj</button>
                    </div>
            </div>
</template>

<script>

const ZOHO = window.ZOHO;

export default {
  name: 'HelloWorld',
  data(){
    return{
      paymentDate: '',
      comments: '',
      paymentMethods: [{text:'Przelew'},{text:'Gotówka'},{text:'Przedpłata'},{text:'Kompensata'}],
      selectedPaymentMethod: "Przelew",
      paymentAmount: 0,
      totalAmountFromDocuments: 0,
      recordList: [],
      alreadyPaidAmount: 0
    }
  },
  methods:{
    settlementDocuments(){
      this.recordList.sort(this.compare)
      this.recordList.forEach(element => this.updateDocuments(element))
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
        this.paymentAmount -= amount;
      }
      if(amount != 0){
        let apiData = new Map();
        let fieldName;
        let checkbox;
        let dateFieldName;
        let commentFieldName;
        let paymentMehodFieldName;
        if(element["Kwota_przelewu_cz_ciowego_1"] == null)
        {
          fieldName = "Kwota_przelewu_cz_ciowego_1";
          checkbox = "Przelew_cz_ciowy_nr_1";
          dateFieldName = "Data_zap_aty_nr_1";
          commentFieldName = "Szczeg_y_przelewu_cz_ciowego_1";
          paymentMehodFieldName = "Spos_b_zap_aty";
        }
        else if(element["Kwota_przelewu_cz_ciowego_2"] == null)
        {
          fieldName = "Kwota_przelewu_cz_ciowego_2";
          checkbox = "Przelew_cz_ciowy_nr_2";
          dateFieldName = "Data_zap_aty_nr_2";
          commentFieldName = "Szczeg_y_przelewu_cz_ciowego_2";
          paymentMehodFieldName = "Spos_b_zap_aty_nr_2";
        }
        else if(element["Kwota_przelewu_cz_ciowego_3"] == null)
        {
          fieldName = "Kwota_przelewu_cz_ciowego_3";
          checkbox = "Przelew_cz_ciowy_nr_3";
          dateFieldName = "Data_zap_aty_nr_3";
          commentFieldName = "Szczeg_y_przelewu_3";
          paymentMehodFieldName = "Spos_b_zap_aty_nr_3";
        }
        else if(element["Kwota_przelewu_cz_ciowego_4"] == null)
        {
          fieldName = "Kwota_przelewu_cz_ciowego_4";
          checkbox = "Przelew_cz_ciowy_nr_4";
          dateFieldName = "Data_zap_aty_nr_4";
          commentFieldName = "Szczeg_y_przelewu_cz_ciowego_4";
          paymentMehodFieldName = "Spos_b_zap_aty_nr_4";
        }
        else if(element["Kwota_przelewu_cz_ciowego_5"] == null)
        {
          fieldName = "Kwota_przelewu_cz_ciowego_5";
          checkbox = "Przelew_cz_ciowy_nr_5";
          dateFieldName = "Data_zap_aty_nr_5";
          commentFieldName = "Szczeg_y_przelewu_cz_ciowego_5";
          paymentMehodFieldName = "Spos_b_zap_aty_nr_5";
        }
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
        let response = await ZOHO.CRM.API.updateRecord(config);
        console.log(response)
      }
    },
    compare(a, b) {
      if (a['Created_Time'] < b['Created_Time'])
          return -1
      if (a['Created_Time'] > b['Created_Time'])
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

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
