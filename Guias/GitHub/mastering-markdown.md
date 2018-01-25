# Microservicio para Asignar Asesores en RTD MX

## Indice

- [Descripción](https://github.com/reynaldoeg/deleteme/blob/master/README.md#descripción)
- [Información Técnica](https://github.com/reynaldoeg/deleteme/blob/master/README.md#información-técnica)
  - [Función getZones()](https://github.com/reynaldoeg/deleteme/blob/master/README.md#getzones)
  - [Función getBranchesOverview()](https://github.com/reynaldoeg/deleteme/blob/master/README.md#getbranchesoverview)
  - [Función assignLeads()](https://github.com/reynaldoeg/deleteme/blob/master/README.md#assignleads)
  - [Función getBranchId()](https://github.com/reynaldoeg/deleteme/blob/master/README.md#getbranchid)
  - [Función fairAssignment()](https://github.com/reynaldoeg/deleteme/blob/master/README.md#fairassignment)
- [Probar de manera local](https://github.com/reynaldoeg/deleteme/blob/master/README.md#probar-de-manera-local)
- [Respuesta](https://github.com/reynaldoeg/deleteme/blob/master/README.md#respuesta)
- [Áreas Involucradas](https://github.com/reynaldoeg/deleteme/blob/master/README.md#Áreas-involucradas)

## Descripción
Este microservicio recibe un archivo json que contiene todas las oportunidades
(leads) a las que se le van a asignar un Asesor de RTD México.

Ejemplo de los parámetros que recibe el microservicio:

```json
{
  "leads": [
    {
      "crm": "Nuevo",
      "system_id": 1,
      "p_status": 1,
      "lead_id": "34f02ac2-0e99-4615-9f16-b0d4ec479711",
      "email": "test180123-1@resuelve.mx",
      "first_name": "Test Uno",
      "postal_code": "56623",
      "state": "Estado de México",
      "assign": 1,
      "borrower_insitute": "Banorte",
      "campaign": "1038157346",
      "come_from": "http://landings.resuelvetudeuda.com",
      "date": "2018-01-23",
      "date_created": "2018-01-23T15:57:44-06:00",
      "debt_amount": 100000,
      "gcid": "2105627000.1516743847",
      "is_update": 1,
      "k_id": "empty",
      "landing": "https://landings.resuelvetudeuda.com/l-oficial",
      "mobile": "5555555555",
      "months_behind": "5",
      "phone": "5555555555",
      "source": "adwordsdisplay",
      "time": "15:57:44"
    },
    {
      ...
    }
  ]
}
```

Los parámetros que son obligatorios son:

- crm
- system_id
- p_status
- lead_id
- email
- first_name
- postal_code
- state

Lo primero que hace el servicio es seleccionar todos los asesores de RTD activos
a los que se le puede asignar un Lead (Candidato), junto con sus sucursales y
topes dinámicos. Todo eso se convierte aun formato que sera usado en la ejecución
de esta función. Con esto:

```sql
SELECT Name, Id, Usuario__c, MD4__c, FechaAsignarOportunidades__c, tope_dinamico__c,
OportunidadesTotalesArd__c, IdSucursal__c, Sucursal__c, Sucursal__r.Name,
Sucursal__r.NumeroDeVendedores__c, AsignarOportunidades__C, NumeroEmpleado__c,
Area__c, Percentage_Occupied__c, Percentage_OccupiedRP__c
FROM RecursosHumanos__c WHERE Status__c = 'Activo' AND AsignarOportunidades__c = true
AND Area__c = 'Ventas' AND Usuario__c != ''
ORDER BY Percentage_Occupied__c ASC
```

Se hace todo en una sola consulta para no  bloquear salesforce con nuestro proceso.

El criterio que toma para seleccionar a los vendedores es que estén **Activos**,
que sean del área de <b>Ventas</b>, que tengan activada la casilla de <b>Asignar oportunidad</b>
y que tengan un <b>usuario asignado</b>.

<p>El script separa a los vendedores por zona y sucursal, además considera a los que
son <b>MD4</b> (candidatos con deuda mayor de $400,000) y los que no lo son.</p>

<p>Tomando en cuenta el código postal y el estado del candidato se selecciona a que
sucursal se asignará, en caso de que todos los vendedores de esa sucursal hayan
alcanzado su tope dinámico mensual de candidatos, las oportunidades se mandarán
a las "Reformas (DF)" y en caso de que también alcancen su tope dinámico mensual,
las oportunidades se mandarán a una sucursal de la zona correspondiente.</p>

<p>Si todos los vendedores alcanzaran su tope dinámico mensual, el candidato se asigna
a la sucursal que le corresponde con fecha de asignación del siguiente mes.</p>

<p>El script regresa los mismos parámetros de cada candidato, y les agrega los
siguientes parámetros:</p>

<ul>
  <li><b>assigned_seller:</b> con el Id de SalesForce del vendedor (Objeto RH).</li>
  <li><b>name_seller:</b> nombre del vendedor asignado.</li>
  <li><b>owner:</b> con el Id de SalesForce del usuario del vendedor (Objeto Users).</li>
  <li><b>branch:</b> nombre de la sucursal del vendedor.</li>
</ul>

<p>Entre los parámetros de cada candidato, puede incluirse el de "<b>assign</b>" que
si es igual a 0 indica que no deberá asignarse ningún vendedor, éste se utiliza
cuando el candidato esté duplicado (ya existe en SalesForce) y sólo se desea
actualizar sus datos sin cambiar de vendedor; en caso de que no incluya dicho
parámetro o que sea igual a 1, el script asignara un vendedor.</p>

<p>Este servicio agrupa las siguientes sucursales y les asigna como si fueran una sóla:</p>

<ul>
  <li>
    <b>Guadalajara:</b>
    <ul>
      <li>Guadalajara 1</li>
      <li>Guadalajara 2</li>
    </ul>
  </li>
  <li>
    <b>Monterrey:</b>
    <ul>
      <li>Monterrey - Lindavista</li>
      <li>Monterrey - Plaza Real</li>
    </ul>
  </li>
  <li>
    <b>Veracruz:</b>
    <ul>
      <li>Veracruz</li>
      <li>Xalapa - Las Animas</li>
    </ul>
  </li>
  <li>
    <b>Reformas:</b>
    <ul>
      <li>Todas las reformas de las oficinas generales</li>
    </ul>
  </li>
</ul>


## Información Técnica

<p>Las principales funciones que componen este servicio son:</p>

### getZones()

#### Descripción

<p>Regresa un array de objetos con la información de cada zona.</p>

<p>Las zonas son:</p>
<ul>
  <li>Mtro y DF</li>
  <li>Centro Sur</li>
  <li>Norte</li>
</ul>

#### Parámetros

<p>Función Callback</p>

#### Valores devueltos

<p>Retorna el array resultante</p>

```sh
[
  {
    "name": "Veracruz",
    "zone": "Centro Sur",
    "id": "Veracruz"
  },
  {
    "name": "Xalapa",
    "zone": "Centro Sur",
    "id": "a0MG000000Cupl0MAB"
  },
  {
    "name": "Iztapalapa",
    "zone": "Mtro y DF",
    "id": "a0MG000000CuplEMAR"
  },
  {
    "name": "Coapa",
    "zone": "Mtro y DF",
    "id": "a0MG000000Cupl4MAB"
  },
  {
    "name": "Cd. Juarez",
    "zone": "Norte",
    "id": "a0MG000000MrsmvMAB"
  },
  {
    "name": "Torreon",
    "zone": "Norte",
    "id": "a0MG000000MpmvHMAR"
  },
  ...

]

```


### getBranchesOverview()

#### Descripción

<p>Esta función es la que obtiene todos los vendedores disponibles y los agrupa por sucursal; la consulta que ejecuta es la siguiente:</p>

```sql
SELECT Name, Id, Usuario__c, MD4__c, FechaAsignarOportunidades__c, tope_dinamico__c,
OportunidadesTotalesArd__c, IdSucursal__c, Sucursal__c, Sucursal__r.Name,
Sucursal__r.NumeroDeVendedores__c, AsignarOportunidades__C, NumeroEmpleado__c,
Area__c, Percentage_Occupied__c, Percentage_OccupiedRP__c
FROM RecursosHumanos__c WHERE Status__c = 'Activo' AND AsignarOportunidades__c = true
AND Area__c = 'Ventas' AND Usuario__c != ''
ORDER BY Percentage_Occupied__c ASC
```

#### Parámetros

<p>Función Callback</p>

#### Valores devueltos

<p>Retorna Un objeto que contiene la información de las sucursales, incluyendo sus vendedores disponibles:</p>

```sh
{
  Guadalajara: {
    salesPersonCount: 8,
    processed: 8,
    idBranch: "Guadalajara",
    branchName: "Guadalajara",
    overflow: false,
    overflowMD4: false,
    salespersons: [
      {
        Id: 'a0UG000000LIBbAMAX',
        Name: 'Victor Flores Muguerza',
        Usuario__c: '005G00000082xM8IAI',
        TopeDinamico: 250,
        OppAsignadas: 90,
        PorcentajeA: 36,
        Sucursal: 'Guadalajara 1'
      },
      {...}
    ],
    salesMD4: [
      {
        Id: 'a0UG000000LIBbAMAX',
        Name: 'Victor Flores Muguerza',
        Usuario__c: '005G00000082xM8IAI',
        TopeDinamico: 250,
        OppAsignadas: 90,
        PorcentajeA: 36,
        Sucursal: 'Guadalajara 1'
      },
      {...}
    ],
    salespersons_index: 0,
    salesMD4_index: 0
  },
  a0MG000000CupknMAB: {
    salesPersonCount: 10,
    processed: 10,
    idBranch: "a0MG000000CupknMAB",
    branchName: "Puebla Galeria Las Animas",
    overflow: false,
    overflowMD4: true,
    salespersons: [
      ...
    ],
    salesMD4: [  
      ...
    ],
    salespersons_index: 0,
    salesMD4_index: 0
  }
  ...

}

```

<p>Los principales elementos del objeto retornado son:</p>

<ul>
  <li><b>salesPersonCount:</b> Número de vendedores por sucursal, registrado en Salesforce</li>
  <li><b>processed:</b> Número de vendedores seleccionados por el servicio (que cumplen las condiciones de asignación).</li>
  <li><b>idBranch:</b> Id de SalesForce de la sucursal (Objeto Sucursal).</li>
  <li><b>branchName:</b> Nombre de la sucursal.</li>
  <li><b>overflowMD4:</b> Indica si todos los vendedores MD4 (VIP) están al tope ( true | false ).</li>
  <li><b>overflow:</b> Indica si todos los vendedores están al tope ( true | false ).</li>
  <li><b>salesMD4:</b> Vendedores VIP (deudas >= $400,000).</li>
  <li>
    <b>salespersons:</b> Vendedores estándar (deudas < $400,000).
    <ul>
      <li><b>Id:</b> con el Id de SalesForce del vendedor (Objeto RH).</li>
      <li><b>Name:</b> nombre del vendedor.</li>
      <li><b>Usuario__c:</b> con el Id de SalesForce del usuario del vendedor (Objeto Users).</li>
      <li><b>TopeDinamico:</b> Máximo número de leads que se le pueden asignar mensualmente.</li>
      <li><b>OppAsignadas:</b> Total de leads asignados en el mes .</li>
      <li><b>PorcentajeA:</b> Porcentaje de leads asignados respecto a su tope .</li>
      <li><b>Sucursal:</b> nombre de la sucursal del vendedor .</li>
    </ul>
  </li>

</ul>

### assignLeads()

#### Descripción

<p>Recorre cada uno de los leads que se pasan a este servicio (event) y les asigna un vendedor dependiendo de su código postal y monto de deuda.</p>

<p>Esta función llama a la función <a href="https://github.com/reynaldoeg/deleteme/blob/master/README.md#getbranchid"> <i>getBranchId()</i></a> pasándole el código postal del lead para obtener el id de la sucursal a la que se va a asignar</p>

<p>Si el Lead tiene una deuda mayor o igual a $400,000 se asignará a vendedores MD4, sino, se asignará a un vendedor estándar.</p>

<p>En caso que el lead sea MD4 y la sucursal no tenga vendedores MD4, se asignará a uno estándar</p>

<p>Toma en cuenta si la sucursal ya tiene todos los vendedores en su tope de asignación y si es así, los envía a las reformas.</p>

<p>Por último llama a la función <a href="https://github.com/reynaldoeg/deleteme/blob/master/README.md#fairassignment"> <i>fairAssignment()</i></a> que es la que se encarga de la asignación.</p>

#### Parámetros

<ul>
  <li>leads</li>
  <li>branches</li>
  <li>Función Callback</li>
</ul>

#### Valores devueltos

<p>Retorna un array con los leads asignados</p>


### getBranchId()

#### Descripción

<p>Regresa el id de SalesForce de la sucursal a la que corresponde el código postal que se pasa como parámetro.</p>

#### Parámetros

<p>
  <b>zipCode: </b> Código postal
</p>

#### Valores devueltos

<p>Retorna el id de la sucursal (String), en caso que sea una sucursal agrupada, regresa el nombre del grupo:</p>

<ul>
  <li>
    <b>Guadalajara:</b>
    <ul>
      <li>Guadalajara 1</li>
      <li>Guadalajara 2</li>
    </ul>
  </li>
  <li>
    <b>Monterrey:</b>
    <ul>
      <li>Monterrey - Lindavista</li>
      <li>Monterrey - Plaza Real</li>
    </ul>
  </li>
  <li>
    <b>Veracruz:</b>
    <ul>
      <li>Veracruz</li>
      <li>Xalapa - Las Animas</li>
    </ul>
  </li>
  <li>
    <b>Reformas:</b>
    <ul>
      <li>Todas las reformas de las oficinas generales</li>
    </ul>
  </li>
</ul>


#### Tabla de asignaciones

<table>
  <tr>
    <th>Sucursal</th>
    <th>Id Sucursal</th>
    <th>CP - desde</th>
    <th>CP - hasta</th>
  </tr>
  <tr>
    <td rowspan="3">San Jerónimo</td>
    <td rowspan="3">a0MG000000OzxsHMAR</td>
    <td>1000</td>
    <td>1990</td>
  </tr>
  <tr>
    <td>5000</td>
    <td>5780</td>
  </tr>
  <tr>
    <td>10000</td>
    <td>10926</td>
  </tr>
  <tr>
    <td rowspan="3">Del Valle</td>
    <td rowspan="3">a0MG000000Cupl3MAB</td>
    <td>4000</td>
    <td>4980</td>
  </tr>
  <tr>
    <td>8000</td>
    <td>8930</td>
  </tr>
  <tr>
    <td>3000</td>
    <td>3940</td>
  </tr>
  <tr>
    <td>Lindavista</td>
    <td>a0MG000000CuplFMAR</td>
    <td>7000</td>
    <td>7990</td>
  </tr>
  <tr>
    <td>Iztapalapa</td>
    <td>a0MG000000CuplEMAR</td>
    <td>9000</td>
    <td>9979</td>
  </tr>
  <tr>
    <td rowspan="2">Coapa</td>
    <td rowspan="2">a0MG000000Cupl4MAB</td>
    <td>13000</td>
    <td>14900</td>
  </tr>
  <tr>
    <td>16000</td>
    <td>16910</td>
  </tr>
  <tr>
    <td>Tijuana</td>
    <td>a0MG000000CuplCMAR</td>
    <td>21000</td>
    <td>22997</td>
  </tr>
  <tr>
    <td>Ciudad Juárez</td>
    <td>a0MG000000MrsmvMAB</td>
    <td>31000</td>
    <td>35987</td>
  </tr>
  <tr>
    <td rowspan="2">León</td>
    <td rowspan="2">a0MG000000Cupl6MAB</td>
    <td>36000</td>
    <td>37699</td>
  </tr>
  <tr>
    <td>38538</td>
    <td>38997</td>
  </tr>
  <tr>
    <td>Querétaro - Plaza San Juan</td>
    <td>a0MG000000CupkoMAB</td>
    <td>37700</td>
    <td>38537</td>
  </tr>
  <tr>
    <td rowspan="4">Metepec</td>
    <td rowspan="4">a0MG000000Cupl5MAB</td>
    <td>50000</td>
    <td>50357</td>
  </tr>
  <tr>
    <td>50400</td>
    <td>50785</td>
  </tr>
  <tr>
    <td>50850</td>
    <td>52497</td>
  </tr>
  <tr>
    <td>52540</td>
    <td>52799</td>
  </tr>
  <tr>
    <td rowspan="2">Satélite</td>
    <td rowspan="2">a0MG000000Cupl2MAB</td>
    <td>52900</td>
    <td>54198</td>
  </tr>
  <tr>
    <td>55700</td>
    <td>55739</td>
  </tr>
  <tr>
    <td rowspan="3">Aragón</td>
    <td rowspan="3">a0MG000000Cupl1MAB</td>
    <td>55000</td>
    <td>55646</td>
  </tr>
  <tr>
    <td>55740</td>
    <td>55778</td>
  </tr>
  <tr>
    <td>56300</td>
    <td>56315</td>
  </tr>
  <tr>
    <td rowspan="8">Monterrey</td>
    <td rowspan="8">Monterrey</td>
    <td>64000</td>
    <td>64997</td>
  </tr>
  <tr>
    <td>65000</td>
    <td>65950</td>
  </tr>
  <tr>
    <td>66000</td>
    <td>66297</td>
  </tr>
  <tr>
    <td>66350</td>
    <td>67298</td>
  </tr>
  <tr>
    <td>67300</td>
    <td>67344</td>
  </tr>
  <tr>
    <td>67350</td>
    <td>67640</td>
  </tr>
  <tr>
    <td>67650</td>
    <td>67344</td>
  </tr>
  <tr>
    <td>67830</td>
    <td>67996</td>
  </tr>
  <tr>
    <td>Guadalajara</td>
    <td>Guadalajara</td>
    <td>44009</td>
    <td>49994</td>
  </tr>
  <tr>
    <td>Puebla Galerías</td>
    <td>a0MG000000CupknMAB</td>
    <td>72000</td>
    <td>75997</td>
  </tr>
  <tr>
    <td>Querétaro Plaza San Juan</td>
    <td>a0MG000000CupkoMAB</td>
    <td>76000</td>
    <td>76998</td>
  </tr>
  <tr>
    <td>Culiacan 3 Rios</td>
    <td>a0MG000000Cupl7MAB</td>
    <td>80000</td>
    <td>82996</td>
  </tr>
  <tr>
    <td>Hermosillo</td>
    <td>a0MG000000CuplBMAR</td>
    <td>83000</td>
    <td>85994</td>
  </tr>
  <tr>
    <td rowspan="2">Veracruz</td>
    <td rowspan="2">Veracruz</td>
    <td>90000</td>
    <td>90754</td>
  </tr>
  <tr>
    <td>91000</td>
    <td>95849</td>
  </tr>
  <tr>
    <td>Reformas</td>
    <td>Reformas</td>
    <td>Todos los</td>
    <td>Demás</td>
  </tr>
</table>


### fairAssignment()

#### Descripción

<p>Se le pasa un Lead y una lista de vendedores a los que se puede asignar, la función ordena los vendedores por el porcentaje de asignación y selecciona el que tenga menor valor.</p>

<p>En caso que el lead que se pasa tenga la propiedad <b>assign</b> = 0 no se le asignará ningún vendedor.</p>

<p>La función verifica que el vendedor no exceda su tope dinámico.</p>

#### Parámetros

<ul>
  <li>lead: Un lead</li>
  <li>salesPeople: Objeto de vendedores a los que se les puede asignar el lead</li>
  <li>forced: (true | false) indica si se asigna en el mes actual o en el siguiente</li>
  <li>Función Callback</li>
</ul>

#### Valores devueltos

<p>Retorna un objeto con las siguientes propiedades:</p>

```sh
{
  lead:{
    crm: "Nuevo",
    system_id: 1,
    p_status: 1,
    ...
    assigned_seller: 'a0UG000000MgcSBMAZ',
    name_seller: 'Uziel Salazar',
    owner: '005G0000008vYVPIA2',
    branch: 'San Jerónimo'
  },
  salesPeople:[
    {
      Id: 'a0U4A00000MJ7Q3UAL',
      Name: 'Victor Soto',
      Usuario__c: '0054A000008yfkMQAQ',
      TopeDinamico: 77,
      OppAsignadas: 43,
      PorcentajeA: 55.84,
      Sucursal: 'San Jerónimo'
    },
    ...
  ],
  assigned: true,
  overflow: false
}
```

<p>Los principales elementos del objeto retornado son:</p>

<ul>
  <li><b>lead:</b> El lead (objeto) que se pasó como parámetro para que se le asignara vendedor, este objeto contiene el vendedor que se le asignó.</li>
  <li><b>salesPeople:</b> Array con la lista completa de los vendedores que se pasaron como parámetro para que se seleccionara al que se asiganría al lead.</li>
  <li><b>assigned (true | false):</b> Indica si fue asignado o no el vendedor.</li>
  <li><b>overflow:</b> Indica si el vendedor llegó o revasó su tope dinámico de asignación.</li>
</ul>






## Probar de manera local

Para poder probar el microservicio, con la línea de comandos, entrar en la
carpeta donde se encuentra

```sh
$ cd appsls/Assignment/asigDinamica
```
Verificar que en el archivo "*event.json*" se encuentren los registros que se
van a insertar (descritos en el punto anterior).

Ejecutar la función con el siguiente comando
```sh
$ sls function run
```


## Respuesta
Una vez llamado el servicio, éste responde con las acciones llevadas acabo, los
parámetros que envía de respuesta son:
**error**: tipo booleano, indica si hubo algún error al ejecutar el servicio (true)
o si se ejecutó exitosamente (false).
**message**: tipo texto, indica el mensaje de la respuesta.
**data**: Muestra el reporte de los registros que se enviaron para asignar, contiene:

 - *num_records_with_advisor*: tipo entero, total de registros asignados.
 - *records_with_advisor*: array,  regresa los mismos datos de cada candidato que
 se enviaron y añade el el parámetro "assigned_seller" con el Id del Asesor que se
 le asignó.

**Nota**: Esta es la respuesta predeterminada de todas las funciones de asignación.
```sh
 {
  "error": false,
  "message": "All leads assigned correctly",
  "data": {
    "num_records_with_advisor": 1,
    "records_with_advisor": [
      {
        "crm": "Nuevo",
        "system_id": 1,
        "p_status": 1,
        "lead_id": "a65dc7f8-6c3c-46da-a030-123abc456def",
        "first_name": "Test Resuelve",
        "email": "test@resuelve.mx",
        "phone": "5555555555",
        "mobile": "5555555555",
        "state": "Distrito Federal",
        "postal_code": "11580",
        "borrower_insitute": "Banco",
        "debt_amount": "35001",
        "months_behind": "6",
        "gcid": "2092563478.1234567890",
        "nani_id": "123456",
        "come_from": "http://m.facebook.com/",
        "source": "facebook",
        "landing": "http://resuelvetudeuda.com/fb",
        "date_created": "2017-05-09T08:15:00-05:00",
        "assigned_seller": "a0UG00000073imX",
        "owner": "005G0000004OlQ0",
        "assignment_date": "2017-05-09T08:20:00-05:00"
      }
    ]
  }
}  

```


### Ejemplo de Respuestas**
Error de Autenticación

``` sh
{
  "error": true,
  "message": "Error on Sales Force login",
  "data": {}
}  

```

Error on Sales Force Query:
``` sh
{
  "error": true,
  "message": "Error on Sales Force Query: SELECT Name, Id, Sucursal__c, tope_dinamico__c
  FROM RecursosHumanos__c WHERE Status__c = 'Activo' AND Area__c = 'Finx'
  AND AsignarOportunidades___c = true AND Sucursal__c != 'a0MG000000Cupkz'
  ORDER BY Sucursal__c ",
  "data": {
    "name": "INVALID_FIELD",
    "errorCode": "INVALID_FIELD"
  }
}  

```


## Áreas Involucradas

 - Marketing.
 - Ventas.
 - Tecnologías de la Información.

## Autores

 - Proyect Manager: Maria Salinas [msalinas@resuelve.mx](msalinas@resuelve.mx)
 - Analista Funcional: Andrés Vera [avera@resuelve.mx](avera@resuelve.mx)
 - Desarrollador: Erick Madrid [emadrid@resuelve.mx](emadrid@resuelve.mx)
 - Desarrollador: Reynaldo Esparza [resparza@resuelve.mx](resparza@resuelve.mx)
