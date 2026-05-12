**core.ts**

export class Item { //cria e exporta a classe "Item"
  private description: string; //Declara um atributo privado para a classe "Item" chamado description.

  constructor(description: string) { //cria um método constructor da classe
    this.description = description; //Salva o valor recebido no atributo da classe.
  } //fecha metodo constructor

  updateDescription(newDescription: string) { //Método para fazer a atualização a descrição do item.
    this.description = newDescription; //Substitui a descrição antiga pela nova.
  } //fecha método

  toJSON() { //transforma o objeto em JSON.
    return { //Retorna um objeto.
      description: this.description //Coloca a descrição atual dentro do objeto JSON.
    } //Fecha o objeto retornado.
  } //Fecha o método toJSON
} //fecha a classe "Item"


export class ToDo { //Cria e exporta a classe "ToDo"
  private filepath: string; //Declara um atributo privado que guarda o caminho do arquivo.
  private items: Promise<Item[]>; //Declara um atributo privado que guarda uma Promise contendo um array de itens.

  constructor(filepath: string) { //Método construtor da classe "ToDo"
    this.filepath = filepath; //Salva o caminho do arquivo recebido no atributo filepath.
    this.items = this.loadFromFile(); //Carrega os itens do arquivo ao iniciar a classe.
  } //fecha método constructor

  private async saveToFile() { //Método privado assíncrono para salvar os itens no arquivo.
    try { //Inicia tratamento de erros.
      const items = await this.items; //Espera os itens serem carregados.
      const file = Bun.file(this.filepath); //Cria referência para o arquivo usando Bun.
      const data = JSON.stringify(items); //Converte os itens para formato JSON.
      return Bun.write(file, data); //Escreve os dados no arquivo.
    } catch (error) { //Captura possíveis erros.
      console.error('Error saving to file:', error); //Exibe erro no console.
    }
  } //fecha método saveToFile

  private async loadFromFile() { //Método privado assíncrono para carregar dados do arquivo.
    const file = Bun.file(this.filepath); //Cria referência ao arquivo.
    if (!(await file.exists())) //Verifica se o arquivo existe.
      return [] //Retorna array vazio caso o arquivo não exista.
    const data = await file.text(); //Lê o conteúdo do arquivo como texto.
    return JSON.parse(data).map((itemData: any) => new Item(itemData.description)); //Transforma o JSON em objetos da classe Item.
  } //fecha método loadFromFile

  async addItem(item: Item) { //Método assíncrono para adicionar um novo item.
    const items = await this.items; //Espera os itens serem carregados.
    items.push(item); //Adiciona o novo item ao array.
    this.saveToFile(); //Salva as alterações no arquivo.
  } //fecha método addItem

  async getItems() { //Método assíncrono para retornar todos os itens.
    return await this.items //Retorna os itens carregados.
  } //fecha método getItems

  async updateItem(index: number, newItem: Item) { //Método assíncrono para atualizar um item.
    const items = await this.items; //Espera os itens serem carregados.
    if (index < 0 || index > items.length) //Verifica se o índice é inválido.
      throw new Error('Index out of bounds'); //Lança erro caso índice seja inválido.
    items[index] = newItem; //Substitui o item antigo pelo novo.
    this.saveToFile(); //Salva as alterações no arquivo.
  } //fecha método updateItem

  async removeItem(index: number) { //Método assíncrono para remover um item.
    const items = await this.items; //Espera os itens serem carregados.
    if (index < 0 || index > items.length) //Verifica se o índice é inválido.
      throw new Error('Index out of bounds'); //Lança erro caso índice seja inválido.
    items.splice(index, 1); //Remove um item do array na posição informada.
    this.saveToFile(); //Salva as alterações no arquivo.
  } //fecha método removeItem

  async findItemByDescription(description: string): Promise<Item | undefined> { //Método assíncrono para buscar item pela descrição.
    const items = await this.items; //Espera os itens serem carregados.
    return items.find(item => item.toJSON().description === description); //Procura item com descrição igual à informada.
  } //fecha método findItemByDescription

  async findItemByIndex(index: number): Promise<Item | undefined> { //Método assíncrono para buscar item pelo índice.
    const items = await this.items; //Espera os itens serem carregados.
    if (index < 0 || index > items.length) //Verifica se o índice é inválido.
      throw new Error('Index out of bounds'); //Lança erro caso índice seja inválido.
    return items[index]; //Retorna o item encontrado na posição informada.
  } //fecha método findItemByIndex
} //fecha classe ToDo



**API Rest e Cliente:**

import todo from "./core.ts"; //Importa o objeto todo do arquivo core.ts.

const server = Bun.serve({ //Cria um servidor HTTP usando Bun.
  port: 3000, //Define a porta 3000 para o servidor.

  routes: { //Define as rotas da aplicação.

    "/api/todo": { //Rota principal da API de tarefas.

      GET: async () => { //Método GET para listar os itens.
        const items = await todo.getItems() //Busca todos os itens.
        return Response.json(items) //Retorna os itens em formato JSON.
      },

      POST: async (req) => { //Método POST para adicionar um item.
        const data = await req.json() as any; //Lê os dados enviados no corpo da requisição.
        const item = data.item || null; //Obtém o item enviado.
        if (!item) //Verifica se o item foi enviado.
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 }); //Retorna erro 400 caso item não exista.
        await todo.addItem(item); //Adiciona o item à lista.
        return Response.json(data); //Retorna os dados recebidos.
      },
    },

    "/api/todo/:index": { //Rota dinâmica que recebe índice do item.

      PUT: async (req) => { //Método PUT para atualizar um item.
        const index = parseInt(req.params.index); //Converte o índice recebido para número inteiro.
        if (isNaN(index)) //Verifica se o índice é inválido.
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 }); //Retorna erro 400.
        
        const data = await req.json() as any; //Lê os dados enviados no corpo da requisição.
        const newItem = data.newItem || null; //Obtém o novo item enviado.

        if (!newItem) //Verifica se o novo item foi enviado.
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 }); //Retorna erro caso não exista.

        try { //Inicia tratamento de erros.
          await todo.updateItem(index, newItem); //Atualiza o item no índice informado.
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`); //Retorna mensagem de sucesso.
        } catch (error: any) { //Captura possíveis erros.
          return Response.json(error.message, { status: 400 }); //Retorna mensagem de erro.
        }
      },

      DELETE: async (req) => { //Método DELETE para remover item.
        const index = parseInt(req.params.index); //Converte o índice para número inteiro.

        if (isNaN(index)) //Verifica se índice é inválido.
          return Response.json('Índice inválido.', { status: 400 }); //Retorna erro 400.

        try { //Inicia tratamento de erros.
          await todo.removeItem(index); //Remove item do índice informado.
          return Response.json(`Item no índice ${index} removido com sucesso.`); //Retorna mensagem de sucesso.
        } catch (error: any) { //Captura possíveis erros.
          return Response.json(error.message, { status: 400 }); //Retorna mensagem de erro.
        }
      },
    },
  },

  async fetch(req) { //Método responsável por servir arquivos estáticos.
    const url = new URL(req.url); //Cria objeto URL da requisição.
    const path = url.pathname; //Obtém o caminho da URL.

    const filePath = (path === '/') //Verifica se o caminho é raiz.
      ? './public/index.html' //Se for "/", carrega index.html.
      : `./public${path}`; //Caso contrário, procura arquivo correspondente na pasta public.

    const file = Bun.file(filePath); //Cria referência ao arquivo.

    if (await file.exists()) { //Verifica se o arquivo existe.
      return new Response(file); //Retorna o arquivo encontrado.
    }

    return new Response(`Not Found`, { status: 404 }); //Retorna erro 404 caso arquivo não exista.
  },
});

console.log(`Server running at http://localhost:${server.port}`); //Exibe no terminal a porta onde o servidor está rodando.
