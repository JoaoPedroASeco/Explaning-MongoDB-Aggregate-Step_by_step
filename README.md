## Video-Aula: https://www.youtube.com/watch?v=pWT-KF0Z2fA&ab_channel=CanaldotNET

  ## Importação de arquivos csv pelo cmd diretamente para o banco MongoDB
  - https://www.mongodb.com/try/download/tools --> download das ferramentas para usar o comando "mongoimport"
  - mongoimport --db DBName --collection collectionName --type=csv --headerline --file=fileName.csv
    ou
  - mongoimport --host 123.123.123.1 --port 1234 -d DB_NAME -c COLLECTION_name --file Name-of-file-to-import
  
  # importando todos os arquivos csv do desafio:
  - mongoimport --db pokemonChallenge --collection combats --type=csv --headerline --file=combats.csv
  - ...

  ## Acessando os dados importados
  - mongo pokemonChallenge -> acessa diretamente o banco de dados pelo nome 
    ou
  - mongo -> entra no terminal do mongo 
  - use DBName -> acessa o banco de dados atravez do terminal do mongo 
  - show collections -> mostra as collections que esse banco de dados possui 
  - db.collectionName.find() --> mostra todos os registros feitos nessa collection --> db.pokemons.find()
  - db.collectionName.find().pretty() --> mostra os dados formatados
  - db.pokemons.find({HP: {$gte: 30}}, {Name: 1, HP: 1, _id: 0})  --> $gte: traz registros da colection que tenham o valor HP maior que 30, e $lt: traz registros da collection que tenha o valor menor que o valor estipulado; {Name: 1, HP: 1} --> traz os campos Name e HP, o numero 1 referenciado nos valores das chaves significa como true, trazendo obrigatoriamente os capos Name e HP e ocultando o campo _id passando seu valor como 0 representando false
  - db.collectionName.count() --> mostra o total de registros feitos nessa collection
  - db.pokemons.find({HP: {$gte: 30}}, {Name: 1, HP: 1, _id: 0}).sort({Name: 1}) --> a função ".sort({ Key: value})" que respectivamente com valor 1, tras do menor -> maior (Ascendente) e -1 do maior -> menor (Descendente)
  - db.pokemons.find({HP: {$gte: 30}}, {Name: 1, HP: 1, _id: 0}).sort({Name: 1}).skip(0).limit(100) --> a função ".skip(number)" ignora uma quantidade de items estipulada no seu valor; a função ".limit(number)" limita a quantidade de registros trazidos na busca apartir do numero estipulado.
  
  ## Oque é Mongo Aggregations Framework 
  - Ele é uma estrutura para agregação de dados modelada no conceito de pipelines de processamento de dados. Os documentos entram em um pipeline de vários estágios, que transforma os documentos em resultados agregados.
  - Trabalha com estágios de pipeline
  - Facilidade em queries mais complexas
  - Permite reutilização de queries
  - Um pipeline sempre trabalha com o resultado anterior

  # pontos a atentar-se
  - A ordem influencia no resultado
  - A ordem incliencia na performance
  - Bastante verboso (dependendo da query, pode ocasionar diversas linhas de código) 

  # 1 - Desafio: Qual é  o nome dos pokemons com a velociade inferior a 60 ?
    - Response: 
      db.pokemons.aggregate([
        { $match: { Speed: { $lt: 60 } } },
        { $project: {
          nome: '$Name',
          hp: '$HP', 
        }}
      ]).pretty()
  
  # 2 - Desafio: Qual é o nome do pokemon, do tipo Fire, que não é um pokemon lendário? Ordene pela força de ataque.
    - Response: 
      db.pokemons.aggregate([
        { $match: { 'Type 1': 'Fire', 'Legendary': 'False' }},
        { $project: {
          nome: '$Name',
          _id: 0,
        }},
        { $sort: { 'Attack': 1 }}
      ]).pretty()

  # 3 - Desafio: Qual é aquantidade de pokémons por tipo?
    - Response: 
      db.pokemons.aggregate([
        { 
          $group: {
            _id: '$Type 1',
            quantidade: { $sum: 1 }
          }
        }
      ]).pretty()

  # 4 - Desafio: Qual é o Nome e HP dos vencedores únicos dos combates realizados?
    - Response: 
      db.combats.aggregate([
        {
          $lookup: { --> faz a busca em uma collection relacionando-se com otra collection
            from: 'pokemons',
            localField: 'Winner',
            foreignField: '#',
            as: 'pokemon'
          }
        },
        { $unwind: '$pokemon'}, --> traz os dados de um pokemon separadamente
        { $project: {
          _id: 0,
          nome: '$pokemon.Name',
          hp: '$pokemon.HP'
        }}
      ]).pretty()

  # 5 - Desafio: Qual é o Nome do pokémon com maio número de vitórias? 
    - Response: 
      db.combats.aggregate([
        {
          $group: {
            _id: '$Winner',
            quantidade: { $sum: 1 },
          }
        },
        {
          $sort: { quantidade: -1 }
        },
        {
          $limit: 1
        },
        {
          $lookup: {
            from: 'pokemons',
            localField: '_id',
            foreignField: '#',
            as: 'pokemon'
          }
        },
        {
          $unwind: '$pokemon'
        },
        {
          $project: {
            _id: 0,
            nome: '$pokemon.Name',
            vitorias: '$quantidade'
          }
        }
      ])

  # 6 - Desafio: A partir dos primeiros 3000 registros, traga o pokemon com maior numer de participações em combates. seguindo qualquer dos dois requisitos abaixo: 
    # Tipo BUG com HP maior ou igual a 70.
    # Tipo Fire com HP maior oi igual a 40.
      - Response: 
        db.combats.aggregate([
          {$limit: 3000},
          {
            $lookup: {
              from: 'pokemons',
              localField: 'First_pokemon',
              foreignField: '#',
              as: 'First_pokemon',
            }
          },
          {
            $lookup: {
              from: 'pokemons',
              localField: 'Second_pokemon',
              foreignField: '#',
              as: 'Second_pokemon',
            }
          },
          {
            $match: {
              $or: [
                { 'First_pokemon.Type 1': 'BUG', 'First_pokemon.HP': { $gte: 70 } },
                { 'Second_pokemon.Type 1': 'Fire', 'Second_pokemon.HP': { $gte: 40 } },
              ]
            }
          },
          {
            $project: {
              items: {
                $concatArrays: ['$First_pokemon', '$Second_pokemon']
              }
            }
          },
          {
            $unwind: '$items'
          },
          {
            $project: {
              nome: '$items.Name',
              tipo: '$items.Type 1',
              id: '$items._id',
            }
          },
          {
            $group: {
              _id: '$nome',
              count: { $sum: 1 }
            }
          },
          {
            $sort: { count: -1 }
          },
          {
            $limit: 1
          }
        ]).pretty()