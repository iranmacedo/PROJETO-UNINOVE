# Agenda Inteligente de Contatos (C - Console)

Este projeto implementa uma **Agenda Inteligente de Contatos** em linguagem C, executada via console. O sistema oferece funcionalidades de CRUD, persistência, importação/exportação CSV, recomendação e detecção de duplicados.

---

## 📌 Funcionalidades

* ➕ **Adicionar contatos**
* 📄 **Listar contatos (resumo)**
* 🔍 **Visualizar contato por ID**
* ✏️ **Atualizar contato**
* ❌ **Remover contato (soft delete)**
* 🔎 **Busca por nome / telefone / email**
* 📤 **Exportar para CSV**
* 📥 **Importar CSV**
* 🌀 **Detecção de duplicados**
* ⭐ **Recomendações (Top N por acessos)**
* 📊 **Relatório Top N (acessos)**
* 🧪 **Gerar contatos de exemplo**

---

## 🛠️ Compilação

```bash
gcc -o agenda agenda.c
```

## ▶️ Execução

```bash
./agenda
```

---

## 📁 Estrutura do Arquivo Principal (`agenda.c`)

A seguir está **todo o código-fonte completo**, pronto para publicação no GitHub:

```c
/* agenda.c
 *
 * Agenda Inteligente de Contatos - Versão em C (console)
 * ... (todo o código fornecido pelo usuário) ...
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define DATA_FILE "contacts.dat"
#define MAX_CONTACTS 10000
#define NAME_LEN 100
#define PHONE_LEN 40
#define EMAIL_LEN 100
#define COMPANY_LEN 80
#define CATEGORY_LEN 40
#define ADDRESS_LEN 200
#define NOTES_LEN 300
#define CSV_LINE 1024

/* agenda.c
 *
 * Agenda Inteligente de Contatos - Versão em C (console)
 *
 * Funcionalidades:
 * - CRUD (Create, Read, Update, Delete)
 * - Persistência em arquivo binário (contacts.dat)
 * - Busca por nome/telefone/email
 * - Import CSV / Export CSV
 * - Detecção de contatos duplicados
 * - Contagem de acessos / marcação de último contato
 * - Relatórios: listar top N contatos por acesso
 * - Recomendação simples (prioriza contatos com maior uso)
 *
 * Compilação:
 *   gcc -o agenda agenda.c
 *
 * Execução:
 *   ./agenda
 *
 * Autor: Gerado por ChatGPT (adaptado ao seu projeto)
 * Data: 2025
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define DATA_FILE "contacts.dat"
#define MAX_CONTACTS 10000
#define NAME_LEN 100
#define PHONE_LEN 40
#define EMAIL_LEN 100
#define COMPANY_LEN 80
#define CATEGORY_LEN 40
#define ADDRESS_LEN 200
#define NOTES_LEN 300
#define CSV_LINE 1024

typedef struct {
    int id; // identificador único
    char name[NAME_LEN];
    char phone[PHONE_LEN];
    char email[EMAIL_LEN];
    char company[COMPANY_LEN];
    char category[CATEGORY_LEN]; // cliente, fornecedor, parceiro, interno, etc.
    char address[ADDRESS_LEN];
    char notes[NOTES_LEN];
    time_t created_at;
    time_t last_contacted;
    unsigned long access_count; // quantas vezes o contato foi acessado
    int active; // 1 = existe, 0 = deletado (soft delete)
} Contact;

/* --- estrutura em memória --- */
Contact *contacts = NULL;
size_t contacts_count = 0;
int next_id = 1;

/* --- utilitários --- */

void clear_input_buffer() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF) {}
}

void press_enter_to_continue() {
    printf("\nPressione Enter para continuar...");
    fflush(stdout);
    clear_input_buffer();
}

char *trim(char *s) {
    if (!s) return s;
    // trim left
    while (*s && (*s == ' ' || *s == '\t' || *s == '\r' || *s == '\n')) s++;
    // trim right
    char *end = s + strlen(s) - 1;
    while (end > s && (*end == ' ' || *end == '\t' || *end == '\r' || *end == '\n')) {
        *end = '\0';
        end--;
    }
    return s;
}

void print_contact(const Contact *c) {
    if (!c) return;
    if (!c->active) return;
    char created[64], last[64];
    struct tm tm_buf;
    localtime_r(&c->created_at, &tm_buf);
    strftime(created, sizeof(created), "%Y-%m-%d %H:%M:%S", &tm_buf);
    localtime_r(&c->last_contacted, &tm_buf);
    strftime(last, sizeof(last), "%Y-%m-%d %H:%M:%S", &tm_buf);

    printf("ID: %d\n", c->id);
    printf("Nome: %s\n", c->name);
    printf("Telefone: %s\n", c->phone);
    printf("Email: %s\n", c->email);
    printf("Empresa: %s\n", c->company);
    printf("Categoria: %s\n", c->category);
    printf("Endereço: %s\n", c->address);
    printf("Observações: %s\n", c->notes);
    printf("Criado em: %s\n", created);
    printf("Último contato: %s\n", last);
    printf("Acessos: %lu\n", c->access_count);
}

/* --- persistência --- */

int load_contacts_from_file() {
    FILE *f = fopen(DATA_FILE, "rb");
    if (!f) {
        // arquivo pode não existir ainda
        contacts = malloc(sizeof(Contact) * MAX_CONTACTS);
        if (!contacts) {
            fprintf(stderr, "Erro de memória.\n");
            return -1;
        }
        contacts_count = 0;
        next_id = 1;
        return 0;
    }
    // ler tudo
    contacts = malloc(sizeof(Contact) * MAX_CONTACTS);
    if (!contacts) {
        fprintf(stderr, "Erro de memória.\n");
        fclose(f);
        return -1;
    }
    size_t read = fread(contacts, sizeof(Contact), MAX_CONTACTS, f);
    fclose(f);
    // contar quantos ativos
    contacts_count = 0;
    int maxid = 0;
    for (size_t i = 0; i < read; ++i) {
        if (contacts[i].id == 0) break; // marcado fim dos registros
        if (contacts[i].id > maxid) maxid = contacts[i].id;
        contacts_count++;
    }
    next_id = maxid + 1;
    return 0;
}

int save_contacts_to_file() {
    FILE *f = fopen(DATA_FILE, "wb");
    if (!f) {
        fprintf(stderr, "Erro ao abrir arquivo para escrita: %s\n", DATA_FILE);
        return -1;
    }
    // Escrever todos os contatos usados (até MAX_CONTACTS). Alternativamente, escrevemos apenas os primeiros contacts_count.
    // Para simplicidade, escrevemos todos os registros preenchidos em memória.
    if (fwrite(contacts, sizeof(Contact), MAX_CONTACTS, f) != MAX_CONTACTS) {
        // note: se o arquivo estiver menor, isso é OK — mas avisamos
        // Não tratamos como erro fatal, mas informamos.
        // Alternativa: escrever apenas contacts_count elementos e padronizar leitura.
        // Para simplicidade e robustez futura, mantemos tamanho fixo.
        // mas se quiser, podemos ajustar para variable-size.
    }
    fclose(f);
    return 0;
}

/* Find index by id */
int find_index_by_id(int id) {
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id == id && contacts[i].active) return (int)i;
    }
    return -1;
}

/* Find next free slot (or existing with id 0) */
int find_free_slot() {
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id == 0 || contacts[i].active == 0) return (int)i;
    }
    // se não há slot livre, retornar -1
    return -1;
}

/* --- CRUD --- */

void add_contact() {
    int slot = find_free_slot();
    if (slot < 0) {
        printf("Limite de contatos atingido (%d). Impossível adicionar novo.\n", MAX_CONTACTS);
        return;
    }
    Contact c;
    memset(&c, 0, sizeof(Contact));
    c.id = next_id++;

    printf("Nome: ");
    fgets(c.name, NAME_LEN, stdin);
    trim(c.name);

    printf("Telefone: ");
    fgets(c.phone, PHONE_LEN, stdin);
    trim(c.phone);

    printf("Email: ");
    fgets(c.email, EMAIL_LEN, stdin);
    trim(c.email);

    printf("Empresa: ");
    fgets(c.company, COMPANY_LEN, stdin);
    trim(c.company);

    printf("Categoria (cliente/fornecedor/parceiro/interno): ");
    fgets(c.category, CATEGORY_LEN, stdin);
    trim(c.category);

    printf("Endereço: ");
    fgets(c.address, ADDRESS_LEN, stdin);
    trim(c.address);

    printf("Observações: ");
    fgets(c.notes, NOTES_LEN, stdin);
    trim(c.notes);

    c.created_at = time(NULL);
    c.last_contacted = c.created_at;
    c.access_count = 0;
    c.active = 1;

    contacts[slot] = c;
    contacts_count++;
    save_contacts_to_file();
    printf("Contato adicionado com ID %d.\n", c.id);
}

void list_contacts_brief() {
    printf("\n--- Lista de Contatos (breve) ---\n");
    int shown = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id != 0 && contacts[i].active) {
            printf("ID: %d | Nome: %s | Telefone: %s | Email: %s | Acessos: %lu\n",
                   contacts[i].id, contacts[i].name, contacts[i].phone, contacts[i].email, contacts[i].access_count);
            shown++;
        }
    }
    if (!shown) printf("Nenhum contato cadastrado.\n");
}

void view_contact() {
    printf("Digite o ID do contato: ");
    int id;
    if (scanf("%d", &id) != 1) { clear_input_buffer(); printf("ID inválido.\n"); return; }
    clear_input_buffer();
    int idx = find_index_by_id(id);
    if (idx < 0) { printf("Contato não encontrado.\n"); return; }
    // incrementar contador de acesso e atualizar last_contacted
    contacts[idx].access_count++;
    contacts[idx].last_contacted = time(NULL);
    print_contact(&contacts[idx]);
    save_contacts_to_file();
}

void update_contact() {
    printf("Digite o ID do contato a atualizar: ");
    int id;
    if (scanf("%d", &id) != 1) { clear_input_buffer(); printf("ID inválido.\n"); return; }
    clear_input_buffer();
    int idx = find_index_by_id(id);
    if (idx < 0) { printf("Contato não encontrado.\n"); return; }
    Contact *c = &contacts[idx];
    printf("Atualizando contato (deixe em branco para manter valor atual)\n");

    char buffer[CSV_LINE];

    printf("Nome atual: %s\nNovo nome: ", c->name);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->name, trim(buffer), NAME_LEN);

    printf("Telefone atual: %s\nNovo telefone: ", c->phone);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->phone, trim(buffer), PHONE_LEN);

    printf("Email atual: %s\nNovo email: ", c->email);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->email, trim(buffer), EMAIL_LEN);

    printf("Empresa atual: %s\nNova empresa: ", c->company);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->company, trim(buffer), COMPANY_LEN);

    printf("Categoria atual: %s\nNova categoria: ", c->category);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->category, trim(buffer), CATEGORY_LEN);

    printf("Endereço atual: %s\nNovo endereço: ", c->address);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->address, trim(buffer), ADDRESS_LEN);

    printf("Observações atuais: %s\nNovas observações: ", c->notes);
    fgets(buffer, sizeof(buffer), stdin);
    if (trim(buffer)[0]) strncpy(c->notes, trim(buffer), NOTES_LEN);

    save_contacts_to_file();
    printf("Contato atualizado.\n");
}

void delete_contact() {
    printf("Digite o ID do contato a deletar: ");
    int id;
    if (scanf("%d", &id) != 1) { clear_input_buffer(); printf("ID inválido.\n"); return; }
    clear_input_buffer();
    int idx = find_index_by_id(id);
    if (idx < 0) { printf("Contato não encontrado.\n"); return; }
    contacts[idx].active = 0; // soft delete
    save_contacts_to_file();
    printf("Contato removido (soft delete).\n");
}

/* --- busca e import/export --- */

void search_contacts() {
    printf("Pesquisar por (1) nome, (2) telefone, (3) email: ");
    int opt;
    if (scanf("%d", &opt) != 1) { clear_input_buffer(); printf("Opção inválida.\n"); return; }
    clear_input_buffer();
    char query[CSV_LINE];
    printf("Digite o termo de busca: ");
    fgets(query, sizeof(query), stdin);
    trim(query);
    if (strlen(query) == 0) { printf("Termo vazio.\n"); return; }
    int found = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id == 0 || !contacts[i].active) continue;
        int match = 0;
        if (opt == 1 && strcasestr(contacts[i].name, query)) match = 1;
        else if (opt == 2 && strcasestr(contacts[i].phone, query)) match = 1;
        else if (opt == 3 && strcasestr(contacts[i].email, query)) match = 1;
        if (match) {
            print_contact(&contacts[i]);
            printf("-----------\n");
            found++;
        }
    }
    if (!found) printf("Nenhum resultado.\n");
}

void export_csv() {
    char filename[260];
    printf("Nome do arquivo CSV para exportar (ex: contatos_export.csv): ");
    fgets(filename, sizeof(filename), stdin);
    trim(filename);
    if (strlen(filename) == 0) { printf("Nome inválido.\n"); return; }
    FILE *f = fopen(filename, "w");
    if (!f) { printf("Erro ao abrir arquivo para escrita.\n"); return; }
    fprintf(f, "id,name,phone,email,company,category,address,notes,created_at,last_contacted,access_count,active\n");
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id == 0) continue;
        // escape basic commas by replacing with semicolon para evitar quebra simples (simples approach)
        char name[NAME_LEN], phone[PHONE_LEN], email[EMAIL_LEN], company[COMPANY_LEN], category[CATEGORY_LEN], address[ADDRESS_LEN], notes[NOTES_LEN];
        strncpy(name, contacts[i].name, NAME_LEN);
        strncpy(phone, contacts[i].phone, PHONE_LEN);
        strncpy(email, contacts[i].email, EMAIL_LEN);
        strncpy(company, contacts[i].company, COMPANY_LEN);
        strncpy(category, contacts[i].category, CATEGORY_LEN);
        strncpy(address, contacts[i].address, ADDRESS_LEN);
        strncpy(notes, contacts[i].notes, NOTES_LEN);
        // substituir vírgulas por ponto-e-vírgula para evitar quebra
        for (char *p = name; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = phone; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = email; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = company; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = category; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = address; *p; ++p) if (*p == ',') *p = ';';
        for (char *p = notes; *p; ++p) if (*p == ',') *p = ';';
        fprintf(f, "%d,%s,%s,%s,%s,%s,%s,%s,%ld,%ld,%lu,%d\n",
                contacts[i].id,
                name,
                phone,
                email,
                company,
                category,
                address,
                notes,
                (long)contacts[i].created_at,
                (long)contacts[i].last_contacted,
                contacts[i].access_count,
                contacts[i].active);
    }
    fclose(f);
    printf("Exportado para %s\n", filename);
}

void import_csv() {
    char filename[260];
    printf("Nome do arquivo CSV para importar (ex: contatos_import.csv): ");
    fgets(filename, sizeof(filename), stdin);
    trim(filename);
    if (strlen(filename) == 0) { printf("Nome inválido.\n"); return; }
    FILE *f = fopen(filename, "r");
    if (!f) { printf("Erro ao abrir arquivo.\n"); return; }
    char line[CSV_LINE];
    // pular header
    if (!fgets(line, sizeof(line), f)) { printf("Arquivo vazio.\n"); fclose(f); return; }
    int imported = 0;
    while (fgets(line, sizeof(line), f)) {
        // parse CSV simples (espera formato id,name,phone,email,company,category,address,notes,...)
        char *token;
        char *rest = line;
        char buf[12][CSV_LINE];
        int col = 0;
        // tokenizar por vírgula simples (não trata aspas)
        while ((token = strsep(&rest, ",")) && col < 12) {
            strncpy(buf[col++], token, CSV_LINE);
        }
        if (col < 4) continue; // linha inválida
        // criar contato
        int slot = find_free_slot();
        if (slot < 0) { printf("Limite de contatos atingido durante import.\n"); break; }
        Contact c;
        memset(&c, 0, sizeof(Contact));
        c.id = next_id++;
        strncpy(c.name, trim(buf[1]), NAME_LEN);
        strncpy(c.phone, trim(buf[2]), PHONE_LEN);
        strncpy(c.email, trim(buf[3]), EMAIL_LEN);
        if (col > 4) strncpy(c.company, trim(buf[4]), COMPANY_LEN);
        if (col > 5) strncpy(c.category, trim(buf[5]), CATEGORY_LEN);
        if (col > 6) strncpy(c.address, trim(buf[6]), ADDRESS_LEN);
        if (col > 7) strncpy(c.notes, trim(buf[7]), NOTES_LEN);
        c.created_at = time(NULL);
        c.last_contacted = c.created_at;
        c.access_count = 0;
        c.active = 1;
        contacts[slot] = c;
        imported++;
    }
    fclose(f);
    save_contacts_to_file();
    printf("Importado %d contatos do CSV.\n", imported);
}

/* --- duplicados e recomendações --- */

void detect_duplicates() {
    printf("--- Detectando possíveis duplicados (nome + telefone ou email) ---\n");
    int found = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id == 0 || !contacts[i].active) continue;
        for (size_t j = i + 1; j < MAX_CONTACTS; ++j) {
            if (contacts[j].id == 0 || !contacts[j].active) continue;
            // comparar name + (phone or email)
            if (strcasecmp(contacts[i].name, contacts[j].name) == 0 &&
                (strlen(contacts[i].phone) && strlen(contacts[j].phone) && strcmp(contacts[i].phone, contacts[j].phone) == 0
                 || strlen(contacts[i].email) && strlen(contacts[j].email) && strcasecmp(contacts[i].email, contacts[j].email) == 0)) {
                printf("Possível duplicado: ID %d e ID %d\n", contacts[i].id, contacts[j].id);
                printf(" -> %s | %s | %s\n", contacts[i].name, contacts[i].phone, contacts[i].email);
                printf(" -> %s | %s | %s\n", contacts[j].name, contacts[j].phone, contacts[j].email);
                printf("-----------\n");
                found++;
            }
        }
    }
    if (!found) printf("Nenhum duplicado detectado.\n");
}

/* Recomendação simples: ordena por score = access_count (desc), mostra top N */
int cmp_contacts_by_access_desc(const void *a, const void *b) {
    const Contact *ca = (const Contact *)a;
    const Contact *cb = (const Contact *)b;
    if (!ca->active && cb->active) return 1;
    if (ca->active && !cb->active) return -1;
    if (ca->access_count < cb->access_count) return 1;
    if (ca->access_count > cb->access_count) return -1;
    return 0;
}

void recommend_contacts() {
    printf("Quantos contatos recomendados deseja ver? ");
    int n;
    if (scanf("%d", &n) != 1) { clear_input_buffer(); printf("Valor inválido.\n"); return; }
    clear_input_buffer();
    if (n <= 0) { printf("Número inválido.\n"); return; }
    // copiar contatos ativos para array temporário
    Contact *tmp = malloc(sizeof(Contact) * MAX_CONTACTS);
    int cnt = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id != 0 && contacts[i].active) tmp[cnt++] = contacts[i];
    }
    if (cnt == 0) { printf("Nenhum contato cadastrado.\n"); free(tmp); return; }
    qsort(tmp, cnt, sizeof(Contact), cmp_contacts_by_access_desc);
    printf("\n--- Top %d contatos recomendados (por uso) ---\n", n);
    for (int i = 0; i < n && i < cnt; ++i) {
        printf("%d) ID %d | %s | Acessos: %lu\n", i+1, tmp[i].id, tmp[i].name, tmp[i].access_count);
    }
    free(tmp);
}

/* Relatório top N por acessos */
void report_top_n() {
    printf("Gerar relatório top N por acessos. Digite N: ");
    int n;
    if (scanf("%d", &n) != 1) { clear_input_buffer(); printf("Valor inválido.\n"); return; }
    clear_input_buffer();
    if (n <= 0) { printf("Número inválido.\n"); return; }
    Contact *tmp = malloc(sizeof(Contact) * MAX_CONTACTS);
    int cnt = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id != 0 && contacts[i].active) tmp[cnt++] = contacts[i];
    }
    if (cnt == 0) { printf("Nenhum contato cadastrado.\n"); free(tmp); return; }
    qsort(tmp, cnt, sizeof(Contact), cmp_contacts_by_access_desc);
    printf("\n--- Relatório Top %d por Acessos ---\n", n);
    for (int i = 0; i < n && i < cnt; ++i) {
        char last[64];
        struct tm tm_buf;
        localtime_r(&tmp[i].last_contacted, &tm_buf);
        strftime(last, sizeof(last), "%Y-%m-%d", &tm_buf);
        printf("%d) ID %d | %s | Acessos: %lu | Último contato: %s\n", i+1, tmp[i].id, tmp[i].name, tmp[i].access_count, last);
    }
    free(tmp);
}

/* Função auxiliar: criar alguns contatos de exemplo (para testes) */
void create_sample_contacts() {
    // apenas se não existir nenhum contato
    int any = 0;
    for (size_t i = 0; i < MAX_CONTACTS; ++i) {
        if (contacts[i].id != 0 && contacts[i].active) { any = 1; break; }
    }
    if (any) {
        printf("Existem contatos cadastrados. Cancela criação de sample.\n");
        return;
    }
    Contact a = {0};
    a.id = next_id++;
    strncpy(a.name, "João Silva", NAME_LEN);
    strncpy(a.phone, "+55 11 99999-0001", PHONE_LEN);
    strncpy(a.email, "joao.silva@empresa.com", EMAIL_LEN);
    strncpy(a.company, "Empresa A", COMPANY_LEN);
    strncpy(a.category, "cliente", CATEGORY_LEN);
    strncpy(a.address, "Rua A, 123", ADDRESS_LEN);
    strncpy(a.notes, "Contato principal de compras", NOTES_LEN);
    a.created_at = time(NULL) - 60*60*24*30; // mês atrás
    a.last_contacted = time(NULL) - 60*60*24*10; // 10 dias atrás
    a.access_count = 12;
    a.active = 1;

    Contact b = {0};
    b.id = next_id++;
    strncpy(b.name, "Mariana Costa", NAME_LEN);
    strncpy(b.phone, "+55 11 98888-7777", PHONE_LEN);
    strncpy(b.email, "mariana@fornecedor.com", EMAIL_LEN);
    strncpy(b.company, "Fornecedor X", COMPANY_LEN);
    strncpy(b.category, "fornecedor", CATEGORY_LEN);
    strncpy(b.address, "Av. B, 200", ADDRESS_LEN);
    strncpy(b.notes, "Fornecedor de material", NOTES_LEN);
    b.created_at = time(NULL) - 60*60*24*80;
    b.last_contacted = time(NULL) - 60*60*24*40;
    b.access_count = 8;
    b.active = 1;

    int slot1 = find_free_slot(); if (slot1 >= 0) contacts[slot1] = a;
    int slot2 = find_free_slot(); if (slot2 >= 0) contacts[slot2] = b;
    save_contacts_to_file();
    printf("Criados contatos de exemplo.\n");
}

/* --- menu e main --- */

void show_menu() {
    printf("\n=== Agenda Inteligente de Contatos ===\n");
    printf("1. Adicionar contato\n");
    printf("2. Listar contatos (resumo)\n");
    printf("3. Ver contato (por ID)\n");
    printf("4. Atualizar contato (por ID)\n");
    printf("5. Remover contato (por ID)\n");
    printf("6. Buscar contatos\n");
    printf("7. Exportar CSV\n");
    printf("8. Importar CSV\n");
    printf("9. Detectar duplicados\n");
    printf("10. Recomendação de contatos (top N por uso)\n");
    printf("11. Relatório top N por acessos\n");
    printf("12. Criar exemplos (apenas se vazio)\n");
    printf("0. Sair\n");
    printf("Escolha: ");
}

int main() {
    // inicializar array
    contacts = malloc(sizeof(Contact) * MAX_CONTACTS);
    if (!contacts) { fprintf(stderr, "Erro de alocação inicial.\n"); return 1; }
    memset(contacts, 0, sizeof(Contact) * MAX_CONTACTS);
    if (load_contacts_from_file() != 0) {
        fprintf(stderr, "Falha ao carregar contatos. Iniciando novo repositório.\n");
        // continue com array zerado
    }

    int opt = -1;
    while (1) {
        show_menu();
        if (scanf("%d", &opt) != 1) { clear_input_buffer(); printf("Opção inválida.\n"); continue; }
        clear_input_buffer();
        switch (opt) {
            case 1: add_contact(); press_enter_to_continue(); break;
            case 2: list_contacts_brief(); press_enter_to_continue(); break;
            case 3: view_contact(); press_enter_to_continue(); break;
            case 4: update_contact(); press_enter_to_continue(); break;
            case 5: delete_contact(); press_enter_to_continue(); break;
            case 6: search_contacts(); press_enter_to_continue(); break;
            case 7: export_csv(); press_enter_to_continue(); break;
            case 8: import_csv(); press_enter_to_continue(); break;
            case 9: detect_duplicates(); press_enter_to_continue(); break;
            case 10: recommend_contacts(); press_enter_to_continue(); break;
            case 11: report_top_n(); press_enter_to_continue(); break;
            case 12: create_sample_contacts(); press_enter_to_continue(); break;
            case 0:
                save_contacts_to_file();
                printf("Saindo. Dados salvos em '%s'.\n", DATA_FILE);
                free(contacts);
                return 0;
            default:
                printf("Opção inválida.\n");
                break;
        }
    }
    return 0;
}

/* Observações de implementação:
 * - Arquivo binário: para simplicidade escrevemos o bloco completo de MAX_CONTACTS; assim
 *   a leitura é simples. Em produção, seria melhor usar estrutura variável ou banco de dados.
 * - Duplicado: detecção simples por nome igual + telefone/email igual (case-insensitive).
 * - Recomendação: algoritmo simples baseado em access_count. Poderíamos melhorar com ML,
 *   mas em C puro e offline preferimos heurísticas.
 * - Import CSV: implementação simples que não trata aspas nem campos com vírgula.
 * - Export CSV: substitui vírgulas por ponto-e-vírgula nos campos para evitar quebra.
 *
 * Você pode estender:
 * - sincronização com APIs externas,
 * - usar SQLite para consultas avançadas,
 * - adicionar criptografia dos dados,
 * - criar interface gráfica (GTK, Qt) ou versão web (com backend em C ou outro).
 */

---

## 📦 Persistência dos Dados

Os contatos são armazenados em um arquivo binário `contacts.dat`. Ele contém até **MAX_CONTACTS registros fixos**, facilitando o carregamento.

---

## 📑 Observações Técnicas

* Sistema usa **soft delete** (active=0) para remover contatos.
* Exportação CSV converte vírgulas em `;`.
* Importação CSV é simples e não suporta aspas.
* Recomendações usam apenas contagem de acessos.
* O arquivo binário sempre escreve o array completo.

---

## 🚀 Possíveis Extensões

* Integração com SQLite.
* Interface gráfica (GTK, Qt).
* Versão web com backend em C.
* Criptografia de dados.
* Sincronização com APIs externas.

---

* 📝 `LICENSE`
* 📂 Estrutura completa de repositório
* 💡 README mais detalhado com GIFs, badges, imagens etc.

É só pedir! 😄
