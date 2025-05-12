#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

//  Lexer 

typedef enum {
    TOK_LET, TOK_FUNCTION, TOK_RETURN,
    TOK_IDENTIFIER, TOK_NUMBER,
    TOK_EQUAL, TOK_PLUS, TOK_MINUS, TOK_STAR, TOK_SLASH,
    TOK_LPAREN, TOK_RPAREN,
    TOK_LBRACE, TOK_RBRACE,
    TOK_COLON, TOK_SEMICOLON,
    TOK_EOF, TOK_UNKNOWN
} TokenType;

typedef struct {
    TokenType type;
    char lexeme[64];
} Token;

#define MAX_TOKENS 256
Token tokens[MAX_TOKENS];
int token_index = 0;

const char *src;
int pos = 0;

void add_token(TokenType type, const char *lexeme) {
    Token tok;
    tok.type = type;
    strncpy(tok.lexeme, lexeme, 63);
    tokens[token_index++] = tok;
    printf("[Lexer] Token: %-12s Lexeme: '%s'\n", (char*[]){
        "LET", "FUNCTION", "RETURN", "IDENTIFIER", "NUMBER",
        "EQUAL", "PLUS", "MINUS", "STAR", "SLASH",
        "LPAREN", "RPAREN", "LBRACE", "RBRACE", "COLON", "SEMICOLON",
        "EOF", "UNKNOWN"
    }[type], lexeme);
}

void lex() {
    while (src[pos]) {
        char c = src[pos];
        if (isspace(c)) {
            pos++;
        } else if (isalpha(c)) {
            int start = pos;
            while (isalnum(src[pos])) pos++;
            char lex[64];
            strncpy(lex, &src[start], pos - start);
            lex[pos - start] = '\0';
            if (strcmp(lex, "let") == 0) add_token(TOK_LET, lex);
            else if (strcmp(lex, "function") == 0) add_token(TOK_FUNCTION, lex);
            else if (strcmp(lex, "return") == 0) add_token(TOK_RETURN, lex);
            else add_token(TOK_IDENTIFIER, lex);
        } else if (isdigit(c)) {
            int start = pos;
            while (isdigit(src[pos])) pos++;
            char lex[64];
            strncpy(lex, &src[start], pos - start);
            lex[pos - start] = '\0';
            add_token(TOK_NUMBER, lex);
        } else {
            char single[2] = { c, '\0' };
            switch (c) {
                case '=': add_token(TOK_EQUAL, single); break;
                case '+': add_token(TOK_PLUS, single); break;
                case '-': add_token(TOK_MINUS, single); break;
                case '*': add_token(TOK_STAR, single); break;
                case '/': add_token(TOK_SLASH, single); break;
                case '(': add_token(TOK_LPAREN, single); break;
                case ')': add_token(TOK_RPAREN, single); break;
                case '{': add_token(TOK_LBRACE, single); break;
                case '}': add_token(TOK_RBRACE, single); break;
                case ':': add_token(TOK_COLON, single); break;
                case ';': add_token(TOK_SEMICOLON, single); break;
                default: add_token(TOK_UNKNOWN, single); break;
            }
            pos++;
        }
    }
    add_token(TOK_EOF, "");
}

//  AST 

typedef enum { AST_NUMBER, AST_BINARY, AST_VAR_DECL } ASTType;

typedef struct ASTNode {
    ASTType type;
    union {
        struct { int value; } number;
        struct { struct ASTNode *left, *right; char op; } binary;
        struct { char name[64]; struct ASTNode *value; } var_decl;
    } data;
} ASTNode;

// Parser 

int current = 0;
Token peek() { return tokens[current]; }
Token advance() { return tokens[current++]; }
int match(TokenType type) { if (peek().type == type) { advance(); return 1; } return 0; }

ASTNode* parse_number() {
    ASTNode *node = malloc(sizeof(ASTNode));
    node->type = AST_NUMBER;
    node->data.number.value = atoi(tokens[current - 1].lexeme);
    printf("[Parser] Parsed number: %d\n", node->data.number.value);
    return node;
}

ASTNode* parse_expression();
ASTNode* parse_primary();
ASTNode* parse_term();
ASTNode* parse_factor();

ASTNode* parse_primary() {
    if (match(TOK_NUMBER)) return parse_number();
    printf("[Parser] Unexpected token: %s\n", peek().lexeme);
    exit(1);
}

ASTNode* parse_term() {
    ASTNode *left = parse_primary();
    while (match(TOK_STAR) || match(TOK_SLASH)) {
        char op = tokens[current - 1].lexeme[0];
        ASTNode *right = parse_primary();
        ASTNode *node = malloc(sizeof(ASTNode));
        node->type = AST_BINARY;
        node->data.binary.left = left;
        node->data.binary.right = right;
        node->data.binary.op = op;
        left = node;  
        printf("[Parser] Parsed term expression (%c)\n", op);
    }
    return left;
}

ASTNode* parse_expression() {
    ASTNode *left = parse_term();
    while (match(TOK_PLUS) || match(TOK_MINUS)) {
        char op = tokens[current - 1].lexeme[0];
        ASTNode *right = parse_term();
        ASTNode *node = malloc(sizeof(ASTNode));
        node->type = AST_BINARY;
        node->data.binary.left = left;
        node->data.binary.right = right;
        node->data.binary.op = op;
        left = node;  
        printf("[Parser] Parsed expression (%c)\n", op);
    }
    return left;
}

ASTNode* parse_var_decl() {
    advance(); // let
    Token id = advance();
    advance(); // =
    ASTNode *value = parse_expression();
    advance(); // ;
    ASTNode *node = malloc(sizeof(ASTNode));
    node->type = AST_VAR_DECL;
    strcpy(node->data.var_decl.name, id.lexeme);
    node->data.var_decl.value = value;
    printf("[Parser] Parsed variable declaration: %s\n", id.lexeme);
    return node;
}

//  Bytecode 

typedef enum { OP_LOAD_CONST, OP_ADD, OP_SUB, OP_MUL, OP_DIV, OP_STORE_VAR } OpCode;

int constants[128];
char variables[128][64];
OpCode bytecode[128];
int const_index = 0, var_index = 0, bc_index = 0;

void emit_bytecode(ASTNode *node) {
    if (node->type == AST_NUMBER) {
        constants[const_index++] = node->data.number.value;
        bytecode[bc_index++] = OP_LOAD_CONST;
        printf("[Bytecode] Emit LOAD_CONST %d\n", node->data.number.value);
    } else if (node->type == AST_BINARY) {
        emit_bytecode(node->data.binary.left);
        emit_bytecode(node->data.binary.right);
        if (node->data.binary.op == '+') {
            bytecode[bc_index++] = OP_ADD;
            printf("[Bytecode] Emit ADD\n");
        } else if (node->data.binary.op == '-') {
            bytecode[bc_index++] = OP_SUB;
            printf("[Bytecode] Emit SUB\n");
        } else if (node->data.binary.op == '*') {
            bytecode[bc_index++] = OP_MUL;
            printf("[Bytecode] Emit MUL\n");
        } else if (node->data.binary.op == '/') {
            bytecode[bc_index++] = OP_DIV;
            printf("[Bytecode] Emit DIV\n");
        }
    } else if (node->type == AST_VAR_DECL) {
        emit_bytecode(node->data.var_decl.value);
        strcpy(variables[var_index++], node->data.var_decl.name);
        bytecode[bc_index++] = OP_STORE_VAR;
        printf("[Bytecode] Emit STORE_VAR %s\n", node->data.var_decl.name);
    }
}

//  Main

int main() {
    src = "let x = 5 * 3;";
    printf("[Main] Source: %s\n", src);
    lex();
    ASTNode *ast = parse_var_decl();
    emit_bytecode(ast);

    printf("\n --Bytecode --\n");
    for (int i = 0; i < bc_index; i++) printf("%d ", bytecode[i]);
    printf("\nConstants: ");
    for (int i = 0; i < const_index; i++) printf("%d ", constants[i]);
    printf("\nVariables: ");
    for (int i = 0; i < var_index; i++) printf("%s ", variables[i]);
    printf("\n");
    return 0;
}
