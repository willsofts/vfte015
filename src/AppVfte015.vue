<!-- App.vue -->
<template>
  <div id="fswaitlayer" class="fa fa-spinner fa-spin"></div>
  <div class="pt-page pt-page-current pt-page-controller search-pager">
    <PageHeader ref="pageHeader" :labels="labels" pid="vfte019" version="1.0.0" showLanguage="true" @language-changed="changeLanguage" :multiLanguages="multiLanguages" :build="buildVersion" :visible="displayPageHeader" />
    <SearchForm ref="searchForm" :labels="labels" :dataCategory="dataCategory" @data-select="dataSelected" @data-insert="dataInsert" />
  </div>
  <teleport to="#modaldialog">
    <EntryForm ref="entryForm" :labels="labels" :dataCategory="dataCategory" @data-saved="dataSaved" @data-updated="dataUpdated" @data-deleted="dataDeleted" />
  </teleport>
</template>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { PageHeader } from '@willsofts/will-control';
import SearchForm from '@/components/SearchForm.vue';
import EntryForm from '@/components/EntryForm.vue';
import { getLabelModel, getMultiLanguagesModel, getMetaInfo } from "@willsofts/will-app";
import { DEFAULT_CONTENT_TYPE, getDefaultLanguage, setDefaultLanguage, getApiUrl } from "@willsofts/will-app";
import { startApplication, serializeParameters, loadAndMergeLabel } from "@willsofts/will-app";

const buildVersion = process.env.VUE_APP_BUILD_DATETIME;
export default {
  components: {
    PageHeader, SearchForm, EntryForm
  },
  setup() {
    const dataChunk = {};
    const dataCategory = {
      tklanguage: [{id: "TH", text: "Thai"},{id: "EN", text: "English"}],
    };
    let labels = ref(getLabelModel());
    let alreadyLoading = ref(false);
    const multiLanguages = ref(getMultiLanguagesModel());
    const displayPageHeader = !(String(getMetaInfo().DISPLAY_PAGE_HEADER)=="false");
    return { displayPageHeader, buildVersion, multiLanguages, labels, dataCategory, dataChunk, alreadyLoading };
  },
  mounted() {
    console.log("App: mounted ...");
    this.$nextTick(async () => {
      //ensure ui completed then invoke startApplication 
      startApplication("vfte015",(data) => {
        console.log("vueapp: message",data);
        if(data.type=="language") {
          let lang = data.language;
          if(lang) {
            this.changeLanguage(lang);
          }
        } else {
          this.multiLanguages = getMultiLanguagesModel();
          this.messagingHandler(data);
          this.loadDataCategories(!this.alreadyLoading,() => {
            this.$refs.pageHeader.changeLanguage(getDefaultLanguage());
          });
          loadAndMergeLabel("vfte015", (success) => {
            if (success) {
              this.changeLanguage(getDefaultLanguage());
            }
          });
        }
      });
    });
  },
  methods: {
    messagingHandler(data) {
      console.log("messagingHandler: data",data); 
    },
    changeLanguage(lang) {
      setDefaultLanguage(lang);
      let labelModel = getLabelModel(lang);
      this.labels = labelModel;
      this.resetDataCategories(lang);
    },
    loadDataCategories(loading,callback) {
      console.log("loadDataCategories: loading",loading);
      if(!loading) return;
      let jsondata = {names: ["tklanguage"]};
      let formdata = serializeParameters(jsondata);
      $.ajax({
        url: getApiUrl()+"/api/category/lists",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("loadDataCategories: error: status",status,"errorThrown",errorThrown);
        },
        success: (data) => {
          this.alreadyLoading = true;
          console.log("loadDataCategories: success",data);
          if(data.body) {
            for(let item of data.body) {
              if(item.category && item.resultset && item.resultset.rows) {
                this.dataChunk[item.category] = item.resultset.rows;
              }
            }
            console.log("data chunk",this.dataChunk);
            this.resetDataCategories();
            if(callback) callback();
          }
        }
      });	
    },
    resetDataCategories(lang) {
      if(!lang) lang = getDefaultLanguage();
      if(!lang || lang.trim().length==0) lang = "EN";
      let tklanguage;
      let tk_category = this.dataChunk["tklanguage"];
      if(tk_category) {
        tklanguage = tk_category.map((item) => { return { id: item.typeid, text: "TH"==lang?item.nameth:item.nameen } });
      }
      if(tklanguage) this.dataCategory.tklanguage = tklanguage;
    },
    dataSelected(item,action) {
      //listen action from search form
      console.log("App: dataSelected",item,"action",action);
      if("edit"==action) {
        this.$refs.entryForm.retrieveRecord({msgcode: item.msgcode, langcode: item.langcode});
      } else if("delete"==action) {
        this.$refs.entryForm.startDeleteRecord({msgdesc: item.msgcode+"-"+item.langtext, msgcode: item.msgcode, langcode: item.langcode});
      }
    },
    dataInsert(filters) {
      //listen action from search form
      console.log("App: record insert",filters);
      this.$refs.entryForm.startInsertRecord();
    },
    dataSaved(data,response) {
      //listen action from entry form when after saved
      console.log("App: record saved");
      console.log("data",data,"response",response);
      this.$refs.searchForm.search();
    },
    dataUpdated(data,response) {
      //listen action from entry form when after updated
      console.log("App: record updated");
      console.log("data",data,"response",response);
      this.$refs.searchForm.search();
    },
    dataDeleted(data,response) {
      //listen action from entry form when after deleted
      console.log("App: record deleted");
      console.log("data",data,"response",response);
      this.$refs.searchForm.search(true);
    },
  }
};
</script>