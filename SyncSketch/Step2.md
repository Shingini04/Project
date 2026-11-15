1. write the signup sgn in create room endpoint
   very naive simple logic rn .
3. writethe middleware the decode the token and gat the create room ep
   use jwt. my http backend has a http secret my ws backend neds to have the same ws scret. config.ts should not have same secrect both http backend and ws backend intead abstract it out somewhere somon. 
4. decode the token for the ws layer as well. send the token to the ws server in a query param for now
5. 
6. initialize a new db package where u write the schema of the project
7. 
8. import the db into the http layer and strt puttin things into the db
9. add a common package where we add the zod schema and the JWT_SECRET
    in packages we have backend-common folder have package.jsona d tscofig.jsonthen in this foldr have another folder src and then index.ts and just the jwt secret here so its shared.
   another folder called common becase itll be used by v=backend and frontnend and introduce zob.in common folder have src folder and then types.ts file and then do createuserschema (what is zod anyway) with username password and name only. and the signinschema with username and password and creareroomschema with name (room name)
