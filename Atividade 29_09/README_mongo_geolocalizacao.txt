ATIVIDADE DE GEOLOCALIZAÇÃO — MONGODB (DSM)
Arquivos:
- inscricoes.json  (30 alunos com campos: nome, curso, turno, bairro, meioTransporte, localizacao GeoJSON)
- pontos.json      (campus + pontos de interesse)

1) Crie um banco (ex.: dsm_geo) e duas coleções:
   use dsm_geo
   db.createCollection("inscricoes")
   db.createCollection("pontos")

2) Importe os dados (no seu terminal/PowerShell, na pasta dos arquivos):
   mongoimport --db dsm_geo --collection inscricoes --file inscricoes.json --jsonArray=false
   mongoimport --db dsm_geo --collection pontos      --file pontos.json      --jsonArray=false

3) Crie os índices geoespaciais:
   db.inscricoes.createIndex({ localizacao: "2dsphere" })
   db.pontos.createIndex({ localizacao: "2dsphere" })

4) Consultas modelo (adapte se desejar):

4.1) 4 alunos mais próximos do CAMPUS:
// ache as coords do campus
const campus = db.pontos.findOne({tipo:"CAMPUS"})
db.inscricoes.aggregate([
  { $geoNear: {
      near: campus.localizacao,
      distanceField: "distancia_m",
      spherical: true
  }},
  { $limit: 4 }
])

4.2) Alunos até 2 km do CAMPUS (≈ 2000 m; raio em radianos = 2000/6378137):
const campus2 = db.pontos.findOne({tipo:"CAMPUS"})
db.inscricoes.find({
  localizacao: {
    $geoWithin: {
      $centerSphere: [ campus2.localizacao.coordinates, 2000/6378137 ]
    }
  }
})

4.3) Agrupar por meioTransporte dos alunos até 3 km do CAMPUS:
const campus3 = db.pontos.findOne({tipo:"CAMPUS"})
db.inscricoes.aggregate([
  { $match: {
      localizacao: {
        $geoWithin: { $centerSphere: [ campus3.localizacao.coordinates, 3000/6378137 ] }
      }
  }},
  { $group: { _id: "$meioTransporte", qtd: { $sum: 1 } } },
  { $sort: { qtd: -1 } }
])

4.4) Próximos 5 alunos de um Ponto de Encontro específico:
const pontoNorte = db.pontos.findOne({nome:"Ponto de Encontro Norte"})
db.inscricoes.aggregate([
  { $geoNear: {
      near: pontoNorte.localizacao,
      distanceField: "distancia_m",
      spherical: true
  }},
  { $limit: 5 }
])

4.5) Rota de van (conceitual): pegue 8 alunos mais próximos do CAMPUS e ordene por distância
const campus4 = db.pontos.findOne({tipo:"CAMPUS"})
db.inscricoes.aggregate([
  { $geoNear: {
      near: campus4.localizacao,
      distanceField: "distancia_m",
      spherical: true
  }},
  { $limit: 8 },
  { $project: { nome: 1, bairro: 1, distancia_m: 1, _id: 0 } }
])

5) Desafio: encontre pares de colegas que moram a <= 800 m um do outro (dica: $function + Haversine ou $geoNear em self-join via $lookup).

Bom estudo!
