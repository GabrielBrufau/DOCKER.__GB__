# practice with node js sequelize postgres docker
*introduccion*
- `mkdir v1.00`

*instalacion*
- `npm init -y`
- `npm i express pg sequelize`

*code*
- `mkdir src > touch index.js`
- `index.js`
```js
const express = require('express');
const app = express();

app.use(express.json());
app.use(express.urlencoded({extended:true});
app.use((req,res,next)=>{
		res.setHeader('Access-Control-Allow-Origin','*');
		res.setHeader('Access-Control-Allow-Methods','GET','PUT','DELETE','POST');
		next();
});

try{
	app.listen(process.env.EXTERNAL_PORT || 3001)
}catch (error){
	console.log(error)
}

```
- now `mkdir controllers routes models utils`
- `cd routes touch dev.js`
- in `dev.js`
```js
const controllers = require('../controllers/dev');
const router = require('express').Router();

router.get('/version',controllers.version);

module.exports = router;
```
- `cd controllers touch dev.js`
- `dev.js`
```js
exports.version = (req,res,next)=>{
			return res.status(200).json("hello")
}
```
- go to `index.js`
```diff
const express = require('express');
const app = express();

app.use(express.json());
app.use(express.urlencoded({extended:true});
app.use((req,res,next)=>{
		res.setHeader('Access-Control-Allow-Origin','*');
		res.setHeader('Access-Control-Allow-Methods','GET','PUT','DELETE','POST');
		next();
});

+	app.use('/dev',require('./routes/dev'));

try{
+	console.log(`
+			server on
+			http://localhost:3001/dev/services {GET:on}
	app.listen(process.env.EXTERNAL_PORT || 3001)
}catch (error){
	console.log(error)
}


```
- Esto ya deberia correr en port 3001
- Ahora creamos el dockerFile
- Go to cd v1.00 and create `touch Dockerfile`
- `Dockerfile`
```Dockerfile
FROM node:12-alpine	/*le dice la version de node que usamos*/

WORKDIR /src		/*en la ruta /src del contenedor*/

COPY package.json package-lock*.json ./ 	/*copiamos los package*/

RUN npm install		/*instalamos*/

COPY . .	/*copiamos todas las carpetas de . a . de docker*/

CMD ["node","src/index.js"]	/*lo que queremos que corra cuando levantemos el contenedor*/
```
- ignoramos el node_modules con .dockerignore
- `touch .dockerignore`
```bash
node_modules

```
- en la terminal construimos la imagen parados en la carpeta donde esta el 
Dockerfile `docker build -t v1.00 .`
- en la terminal levantamos el contenedor `docker run -p 3001:30001 v1.00`
- `Ctrl c` para parar el contenedor o `docker stop nameofconteiner o idconteiner`
- usando docker-compose.yml
- in `cd v1.00` `touch docker-compose.yml`
```docker
/*version del compose schema*/
version:
	"3.7"
/*next,we'll define the list of services or conteiners we want to run as part of our aplication*/
services:
	src:
		container_name:
			v1.00
		image:
			v1.00:docker-compose
		build:
			context: .
		ports:
			- "3001:3001"
		environment:
			- EXTERNAL_PORT=3001
```
- run in terminal `docker-compose build`
- run in terminal `docker-compose up` levanta el contenedor

- tratemos de conectar postgres
- in docker-compose.yml
```diff
services:
	src:
		container_name: v1.00
		image: v1.00:docker-compose
		build:
			context: .
		ports:
			- "3001:3001"
		environment:
			- EXTERNAL_PORT=3001
+		depends_on:
+			- db
+	db:
+		container_name: name_conteiner_postgres
+		image: "postgres:12"
+		port:
+			- "5432:5432"
+		environment:
+			- POSTGRES_USER=postgres
+			- POSTGRES_PASSWORD=12345
+			- POSTGRES_DB=name_db_docker_postgres
+		volumes:
+			- nps_data:/var/lib/postgresql/data
+			- ./node_modules
+volumes:
+	nps_data: {} 

```
- `docker-compose up`
- deberia levantar el servidor y la db
- en otra terminal podriamos interactuar con la db
- `docker exec -it iddelcontenedordepostgres psql -U nameuserpostgres namebasedatos`

- conectemos la db con la app
- go to `cd utils/db.js`
```js
const Sequelize = require('sequelize');

const sequelize = new Sequelize(
	process.env.POSTGRES_DB,
	process.env.POSTGRES_USER,
	process.env.POSTGRES_PASSWORD,
	{
	host:process.env.POSTGRES_HOST,
	dialect:'postgres'
	}
);
module.exports = sequelize;
```
- ahora creamos las environment en docker-compose.yml
```diff
services:
	src:
		container_name: v1.00
		image: v1.00:docker-compose
		build:
			context: .
		ports:
			- "3001:3001"
		environment:
			- EXTERNAL_PORT=3001
+			- POSTGRES_DB=db_postgres
+			- POSTGRES_USER=postgres
+			- POSTGRES_PASSWORD=12345
+			- POSTGRES_HOST=db
		depends_on:
			- db
	db:
		container_name: DB_postgres
		image: "postgres:12"
		port:
			- "5432:5432"
		environment:
			- POSTGRES_USER=postgres
			- POSTGRES_PASSWORD=12345
			- POSTGRES_DB=db_postgres
		volumes:
			- nps_data:/var/lib/postgresql/data
			- ./node_modules
volumes:
	nps_data: {}
```
- now go to `cd routes/users.js`
```js
const controllers = requiere('../controllers/users.js');

const router = require('express').Router();

//crud
router
	.get('/',controllers.getAll)
	.get('/:id',controllers.getOne)
	.post('/',controllers.createOne)
	.put('/:id',controllers.updateOne)
	.delete('/:id',controllers.deleteOne);

module.exports = router;



```
- definamos los schemas en models para usarlos en los controllers
- `cd models/users.js`
```js
const Sequelize = require('sequelize');
const db = require('../utils/db.js');

const User = db.define('users',{
	id:{
		type:Sequelize.INTEGER,
		autoIncrement:true,
		allowNull:false,
		primaryKey:true
	},
	username:{
		type:Sequelize.STRING,
		allowNull:false,
		unique:true
	},
	email:{
		type:Sequelize.STRING,
		allowNull:false
	},
	password:{
		type:Sequelize.STRING,
		allowNull:false
	}
});

module.exports = User;
```
- now go to `cd controllers/users.js`
```js
const User = require('../models/users.js');

exports.getAll = async (req,res,next)=>{
	try{
		const ALL = await User.findAll();
		return res.status(200).json(ALL);
	}catch(error){
		return res.status(500).json(error);
	}
}
```
- now in src/index.js
```diff
const express = require('express');
const app = express();
+	const sequelize = require('./utils/db.js');
+	const User = require('./models/users.js');

app.use(express.json());
app.use(express.urlencoded({extended:true});
app.use((req,res,next)=>{
		res.setHeader('Access-Control-Allow-Origin','*');
		res.setHeader('Access-Control-Allow-Methods','GET','PUT','DELETE','POST');
		next();
});

app.use('/dev',require('./routes/dev'));
+	app.use('/users',require('./routes/users'));

+	(async ()=>{
try{
+	await sequelize.sync({force: false});
console.log(`
	server on
	http://localhost:3001/dev/services {GET:on}`)
app.listen(process.env.EXTERNAL_PORT || 3001)
}catch (error){
	console.log(error)
}
+	})()
```
- now run in ubuntu `sudo docker-compose up --build` --build le dice
que construlla los contenedores con los cambios 

- una vez que estan conectados y corriendo podemos crear los otros controllers
- go to `cd controllers/users.js`
```diff
const User = require('../models/users.js');

exports.getAll = async (req,res,next)=>{
	try{
		const ALL = await User.findAll();
		return res.status(200).json(ALL);
	}catch(error){
		return res.status(500).json(error);
	}
}

+exports.getOne = async (req,res,next)=>{
+        try{
+                const USER = await User.findByPk(req.params.id);
+                return res.status(200).json(USER);
+        }catch (error){
+                return res.status(500).json(error);
+        }
+}

+exports.createOne = async (req,res,next)=>{
+        try{
+                const MODEL_USER = {
+                        username:req.body.username,
+                        email:req.body.email,
+                        password:req.body.password
+                }
+                try{
+                        const USER_CREATED = await User.create(MODEL_USER);
+                        console.log('console# user created',USER_CREATED);
+                        return res.status(201).json(USER_CREATED);
+                }catch (error){
+                return res.status(500).json(error);
+                }
+        }catch (error){
+                return res.status(500).json(error)
+        }
+}

+exports.updateOne = async (req,res,next)=>{
 +       try{
  +              const USER_MODEL = {
   +                     username:req.body.username,
    +                    email:req.body.email,
            +            password:req.body.password
           +     };
          +      try{
         +       const USER_UPDATED = await User.update(USER_MODEL,{where:{id:req.params.id}});
        +        console.log('console# user update',USER_UPDATED)
       +         return res.status(200).json(USER_UPDATED);
      +          }catch (error){
     +                   return res.status(500).json(error);
    +            }
   +     }catch (error){
  +              return res.status(500).json(error);
 +       }
+}

+exports.deleteOne = async (req,res,next)=>{
       + try{
      +          const USER_DELETED = await User.destroy({where:{id:req.params.id}});
     +           console.log('console# user deleted',USER_DELETED);
    +            return res.status(200).json(USER_DELETED);
   +     }catch (error){
  +              return res.status(500).json(error);
 +       }
+}

```
- run in ubuntu `sudo docker-compose up --build`








