<template>
  <div>
    <div class="card">
      <pv-toolbar class="mb-4">
        <template #start>
          <pv-button label="New" icon="pi pi-plus" class="p-button-success mr-2" @click="openNew"/>
          <pv-button label="Delete" icon="pi pi-trash" class="p-button-danger" @click="confirmDeleteSelected"
                      :disabled="!selectedTutorials || !selectedTutorials.length"/>
        </template>
        <template #end>
          <pv-button label="Export" icon="pi pi-download" class="p-button-help" @click="exportToCSV($event)"/>
        </template>
      </pv-toolbar>
    </div>

    <pv-data-table
      ref="dt"
      :value="tutorials"
      v-model:selection="selectedTutorials"
      datakey="id"
      :paginator="true"
      :rows="10"
      :filters="filters"
      paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
      :rowsPerPageOptions="[5, 10, 15]"
      currentPageReportTemplate="Showing {first} to {last} of {totalRecords} tutorials"
      responsiveLayout="scroll"
    >
      <template #header>
        <div class="table-header flex flex-column md:flex-row md:justify-content-between">
          <h5 class="mb-2 md:m-0 p-as-md-center text-xl">Manage Tutorials</h5>
          <span class="p-input-icon-left">
              <i class="pi pi-search"/>
              <pv-input-text v-model="filters['global'].value" placeholder="Search..."/>
            </span>
        </div>
      </template>

      <pv-column selectionMode="multiple" style="width: 3rem" :exportable="false"></pv-column>
      <pv-column field="id" header="id" :sortable="true" style="min-width: 12rem"></pv-column>

    </pv-data-table>

  </div>
</template>

<script>
import { TutorialsApiService } from "../services/tutorials-api.service";
import {FilterMatchMode} from "primevue/api";

export default {
  //no te olvides borrar el ".component"
  name: "tutorial-list",
  //siempre empezar por la lógica
  data() {
    return {
      tutorials: [],
      //para una operación típica de crud vamos a tener 2 cosas
      //un botón "new" y en cada fila de la tabla vamos a tener un botón "editar" esa fila en particular
      //el agregar un elemento o el editar un elemento nos llevan al mismo diálogo porque los datos son los mismos
      //la única diferencia en "new" los campos están vacíos y en "editar" los campos tienen el valor de la fila actual
      //y los diálogos se configuran en el template y están a la espera para aparecer se asocian con un modelo "booleano"
      //si es true el diálogo aparece y si es false desaparece
      tutorialDialog: false,
      deleteTutorialDialog: false,
      //esto lo dejamos puesto porseaca
      //aún no ha dicho que hace
      deleteTutorialsDialog: false,
      //una VARIABLE que representa el tutorial que se está seleccionando ahorita
      tutorial: {},
      //dejemos abierto la posibilidad de seleccionar tutorials
      selectedTutorials: null,
      //esto me parece que es para los filtros de la tabla,
      //pero aún no lo ha dicho
      filters: {},
      //para confirmar si el usuario de verdad quiere hacer la acción
      submitted: false,
      //esto me parece que es mi parte para saber en qué estado se encuentra
      statuses: [
        //esto porque en la base de datos hay un booleano de este tipo para esta entidad
        //ojito esto creo que es mi parte
        { label: "Published", value: "published" },
        { label: "Unpublished", value: "unpublished" },
      ],
      //¿esto para qué es?
      tutorialsService: null,
    };
  },
  //Lo siguiente sería
  //ESTO SOLO ES PARA POBLAR LA DATA
  //PARA LA LÓGICA VA EN LA PARTE DE "MÉTODOS"
  created() {
    //esto me parece que es un constructor
    //NOTA: utiliza ese "tutorialsService" que tiene valor null
    this.tutorialsService = new TutorialsApiService();
    //quisieramos que vaya cargando un poco la información
    //aca en el componente si corresponde tratar la llegada del response
    this.tutorialsService.getAll().then((response) => {
      //ahora lo vamos a poblar con la llegada del response
      //porqué lo hacemos de esta forma?
      //ya que axios ya devuelve el "data" según la clase que estas usando en este caso "getAll"
      this.tutorials = response.data();
      //DETALLE: hacerlo un poco más amigable para el usuario
      //qué esta haciendo esto
      //simplemente cambia el valor de "published" de la entidad "tutorials"
      //cambia "true/false" por "Published/Unpublished"
      this.tutorials.forEach(
        (tutorial) =>
          (tutorial.published = tutorial.published
            ? "Published"
            : "Unpublished")
      );
      console.log("created");
    });
    //TODO: Init Filters
    this.initFilters();
  },
  //ahora vamos a utilizar un poco de métodos
  //empezamos a pensar cómo va a interactuar el usuario
  //qué cosas va a hacer el usuario
  methods: {
    //CREAR UNA VERSIÓN VISUALIZABLE DEL TUTORIAL
    getDisplayableTutorial(tutorial) {
      tutorial.status = tutorial.published
        ? this.statuses[0].label
        : this.statuses[1].label;
      return tutorial;
    },
    //establecer el paso inverso
    //una cosa es lo que se ve y otra es lo que se almacena
    //no podemos enviar al API un objeto que tengo un atributo status
    //hacemos una versión que el API estaría esperando
    getStorableTutorial(displayableTutorial) {
      return {
        id: displayableTutorial.id,
        title: displayableTutorial.title,
        description: displayableTutorial.description,
        published: displayableTutorial.status.label === "Published",
      };
    },

    //ahora vamos a hacer la funcionalidad de agregar un nuevo "tutorial"
    //qué cosa queremos que paso cuando el usuario hace click en el botón "new"
    //como es nuevo nuestra variable "tutorial" debería ponerse vacio para recibir un elemento vacio
    //la variable "submitted" que indica si el usuario confirmo o no el nuevo elemento debería estar en "false" hasta que le de click
    //se presente el "diálogo" para agregar elemento que es el mismo que editar
    openNew() {
      //inicializamos esta variable para que este lista para recibir un elemento nuevo
      this.tutorial = {};
      //inicializamos a variable para que este lista cuando el usuario la use
      this.submitted = false;
      //esto esta en true porque se debe mostrar el dialog cuando se utilize el openNew()
      this.tutorialDialog = true;
    },
    //ahora ocultamos el dialog y colocamos la variable submitted en false "ya que en el template ha cambiado"
    hideDialog() {
      this.tutorialDialog = false;
      this.submitted = false;
    },
    //estamos combinando 2 mundos ya que por un lado tenemos la data en tipo en json y que se identifica por id
    //y por el otro lado tenemos la tabla en donde la data se identifica por el indice entonces
    //¿Cómo unimos estos 2 mundos?
    //pues con esta clase
    findIndexById(id) {
      //aca estamos identificando de datos por índice a datos por id
      //lo que quiero que hagas es
      //encuentres de la tabla "finIndex" el elemento "tutorial" que tengo el id igual al que le doy "id"
      return this.tutorials.findIndex((tutorial) => tutorial.id === id);
    },
    //recordemos tenemos un "tutorialDialog" el cual se usa para agregar y editar
    //ya que una vez el usuario presione el botón "guardar" se va a mostrar este dialog
    //entonces el "guardar" va a ser una acción
    //y tengo que preguntar si estoy agregando o editando
    saveTutorial() {
      //el usuario ha presionado el botón de guardar y ha llamado a la clase "saveTutorial"
      this.submitted = true;
      //Primero tengo que verificar si tiene título, luego de eliminar los espacios le queda valor entonces entro a trabajar
      //esta variable tutorial sirve para guardar el elemento seleccionado
      //pero ¿Dónde le dices esa lógica?
      //me parece que es en el template
      if (this.tutorial.title.trim()){
        //ahora con este if estoy discriminando entre agregar y editar
        //ya que si tiene valor de id estoy editando
        //si no tiene id estoy agregando
        if (this.tutorial.id) {
          this.tutorial = this.getStorableTutorial(this.tutorial);
          //NOTA: cada vez que quieras utilizar los servicios del componente se hace a través de esta variable "tutorialsService"
          //con esta linea le decimos que actualize el elemento y le pasamos el id y el tutorial
          this.tutorialsService.update(this.tutorial.id, this.tutorial).then((response) => {
            //ahora que ya he editado la data del backend cuando llegue el response
            //necesito reflejar ese cambio en la tabla y como es un arreglo el elemento tiene la fila con el valor anterior
            //ahora aca usamos la clase antes creada "findIndexById"
            //y como queremos tenerlo como un elemento lo hacemos entre []
            //y lo chancamos en la variable que sirve para identificar al elemento actualmente seleccionado
            this.tutorials[this.findIndexById(response.data.id)] = this.getDisplayableTutorial(response.data);
            //ahora toco componente tiene inherente tiene un "toast"
            //esto sirve para confirmar si se realizo el registro es un componente de PrimeVue
            this.$toast.add({severity: "success", summary: "Successful", detail: "Tutorial updated", life: 3000});
            //PARA QUÉ SIRVE EL CONSOLE.LOG??????
            console.log(response);
          });
        } else {
          //ahora desde este else es para agregar
          this.tutorial.id = 0;
          console.log(this.tutorial);
          this.tutorial = this.getStorableTutorial(this.tutorial);
          this.tutorialsService.create(this.tutorial).then((response) => {
            this.tutorial = this.getDisplayableTutorial(response.data);
            this.tutorials.push(this.tutorial);
            this.$toast.add({severity: "success", summary: "Successful", detail: "Tutorial created", life: 3000});
            //PARA QUÉ SIRVE EL CONSOLE.LOG??????
            console.log(response);
          });
        }
        this.tutorialDialog = false;
        this.tutorial = {};
      }
    },
    //métodos que necesitamos
    editTutorial(tutorial) {
      console.log(tutorial);
      this.tutorial = {...tutorial};
      console.log(this.tutorial);
      this.deleteTutorialDialog = true;
    },
    deleteTutorial() {
      this.tutorialsService.delete(this.tutorial.id).then((response) => {
        this.tutorials = this.tutorials.filter(
          (t) => t.id !== this.tutorial.id
        );
        this.deleteTutorialDialog = false;
        this.tutorial = {};
        this.$toast.add({severity: "success", summary: "Successful", detail: "Tutorial deleted", life: 3000});
        //PARA QUÉ SIRVE EL CONSOLE.LOG??????
        console.log(response);
      });
    },
    //aprovechemos una funcionalidad que tiene el dataTable de primevue para exportar el contenido de la tabla como csv
    exportToCSV() {
      //dt es como lo vamos a referenciar en el template
      //esto lo vamos a asociar con un botón
      this.$refs.dt.exportToCSV();
    },
    //pueda seleccionar más de un registro y puedo eliminarlos
    confirmDeleteSelected() {
      this.deleteTutorialsDialog = true;
    },
    //eliminación de varios tutorials
    deleteSelectedTutorials() {
      this.selectedTutorials.forEach((tutorial) => {
        this.tutorialsService.delete(tutorial.id).then((response) => {
          this.tutorials = this.tutorials.filter(
            (t) => t.id !== this.tutorial.id
          );
          console.log(response);
        });
      });
      this.deleteTutorialsDialog = false;
    },
    //para inicializar el filtro
    //este tenemos que invocarlo en el created
    initFilters() {
      this.filters = {
        global: { value: null, matchMode: FilterMatchMode.CONTAINS },
      };
    },
  },
};
</script>

<style scoped></style>
