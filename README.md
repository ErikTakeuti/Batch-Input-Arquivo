# Batch-Input-Arquivo

### Batch Input com Arquivo

```abap

REPORT zprog_batchinput_bas_02 LINE-SIZE 172 LINE-COUNT 65.
"LINE SIZE especifica o número de letras que podem ser acomodadas em uma linha. se o número de caracteres for maior que isso,
"uma nova linha será acionada automaticamente

"LINE COUNT especifica o número de linhas que podem caber em uma página, se o número de linhas for maior que isso,
"uma nova página é acionada automaticamente

"Eles afetam a exibição da saída.

**********************************************************************

*********************************
***  Definição de Parâmetros  ***
*********************************

SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-001.

PARAMETERS: p_modo(1) TYPE c DEFAULT 'N'.

SELECTION-SCREEN: ULINE.

PARAMETERS: p_file TYPE string DEFAULT 'c:clientes9.txt' OBLIGATORY.

SELECTION-SCREEN END OF BLOCK b1.

*************************************
*** Variáveis Globais do Programa ***
*************************************

DATA:
  w_contatot  TYPE i, "Total de Registros do Arquivo
  w_contaval  TYPE i, "Registros Válidos
  w_containv  TYPE i, "Registros Inválidos
  w_contapro  TYPE i, "Registros processados em Call Trans
  w_contabdc  TYPE i, "Registros postos em pasta BDC
  w_numrec    TYPE i, "Número do Registro corrente
  w_numerros  TYPE i, "Contador de Erros de Call Transac
  w_codmsg(5) TYPE i, "Código de Mensagem na Forma IINNN
  w_valido(1) TYPE i, "Flag para Validação de Registros
  w_conta     TYPE i, "Calculos de uso geral
  w_subrc     LIKE sy-subrc.

*********************************
*** Tabelas Internas Globais  ***
*********************************

*** Tabela com Definição das Transações Batch Input
DATA: BEGIN OF t_bdc OCCURS 10.

    INCLUDE STRUCTURE bdcdata.

DATA: END OF t_bdc.


*Tabela para Registro de Mensagens de Erro das Transações
DATA: BEGIN OF t_msg OCCURS 10.

    INCLUDE STRUCTURE bdcmsgcoll.

DATA: END OF t_msg.

************************************************************************
***                          TYPES                                   ***
************************************************************************

TYPES: BEGIN OF ty_entrada,

         kunnr(4)  TYPE c, "Código do cliente
         vkorg(4)  TYPE c, "Organização de vendas
         vtweg(2)  TYPE c, "Canal de distribuição
         spart(2)  TYPE c, "Setor de Atividade
         ktokd(4)  TYPE c, "Grupo de Contas
         name1(18) TYPE c, "Nome do Cliente
         land1(2)  TYPE c, "País
         spras(2)  TYPE c, "Idioma

       END OF ty_entrada.

************************************************************************
*****               TABELAS INTERNAS E ESTRUTURAS                    ***
************************************************************************

DATA:
  t_entrada TYPE TABLE OF ty_entrada,
  e_entrada TYPE ty_entrada.

*************************************
***  Evento: at selection screen  ***
*************************************

AT SELECTION-SCREEN.
  "evento que permite manipular ações do usuário e do sistema
  "durante o processo de exibição/alteração da tela de seleção de reports.

  IF NOT p_modo CA 'AEN'.

    MESSAGE e000(zcpx001_13) WITH TEXT-m01.

  ENDIF.

**********************************
*** Evento start-of-selection  ***
**********************************

START-OF-SELECTION.

* Rotina para lelitura de arquivo
  PERFORM le_arquivo.

* Rotina para processamento dos dados de Batch Input
  PERFORM processa.

* Rotina para executar Batch Input
  PERFORM executa.

***********************************************************
*** Subrotina genérica para adicionar telas em bdcdata  ***
***********************************************************

FORM bdc_tela USING programa tela.

  CLEAR t_bdc.
  t_bdc-program  = programa.
  t_bdc-dynpro   = tela.
  t_bdc-dynbegin = 'X'.
  APPEND t_bdc.

ENDFORM. "BDC_TELA

FORM bdc_campo USING campo valor.

  CLEAR t_bdc.
  t_bdc-dynpro = space.
  t_bdc-fnam   = campo.
  t_bdc-fval   = valor.
  APPEND t_bdc.

ENDFORM. "BDC_CAMPO

***********************************
***   Processa registro no R/3  ***
***********************************

FORM processa.

  REFRESH t_bdc.

  LOOP AT t_entrada INTO e_entrada.

*Tela inicial da Transação
    PERFORM bdc_tela  USING 'SAPMF02D' '0107'.
    PERFORM bdc_campo USING 'BDC_OKCODE' '/00'.
    PERFORM bdc_campo USING 'RF02D-KUNNR' e_entrada-kunnr.
    PERFORM bdc_campo USING 'RF02D-VKORG' e_entrada-vkorg.
    PERFORM bdc_campo USING 'RF02D-VTWEG' e_entrada-vtweg.
    PERFORM bdc_campo USING 'RF02D-SPART' e_entrada-spart.
    PERFORM bdc_campo USING 'RF02D-KTOKD' e_entrada-ktokd.

*Tela de Dados principais
    PERFORM bdc_tela  USING 'SAPMF02D' '110'.
    PERFORM bdc_campo USING 'KNA1-NAME1'   e_entrada-name1.
    PERFORM bdc_campo USING 'KNA1-LAND1'   e_entrada-land1.
    PERFORM bdc_campo USING 'KNA1-SPRAS'   e_entrada-spras.
    PERFORM bdc_campo USING 'BDC_OKCODE' '=UPDA'.

*Tela de Dados principais
    PERFORM bdc_tela  USING 'SAPMF02D' '315'.
    PERFORM bdc_campo USING 'BDC_OKCODE' '=UPDA'.
    PERFORM bdc_campo USING 'KNVV-KZAZU' 'X'.
    PERFORM bdc_campo USING 'KNVV-VSBED' '01'.
    PERFORM bdc_campo USING 'KNVV-ANTLF' '9'.

*Tela de Dados principais
    PERFORM bdc_tela  USING 'SAPMF02D' '1350'.
    PERFORM bdc_campo USING 'KNVI-TAXKD(01)' '0'.

    ENDLOOP.
ENDFORM. "PROCESSA

*&---------------------------------------------------------------------*
*&      Form  LE_ARQUIVO - LÊ ARQUIVO DE TEXTO
*&---------------------------------------------------------------------*

FORM le_arquivo .

  CALL FUNCTION 'GUI_UPLOAD'
    EXPORTING
     filename                      = p_file
     filetype                      = 'ASC'
    TABLES
     data_tab                      = t_entrada.

  IF sy-subrc <> 0.
 MESSAGE ID SY-MSGID TYPE SY-MSGTY NUMBER SY-MSGNO
         WITH SY-MSGV1 SY-MSGV2 SY-MSGV3 SY-MSGV4.
  ENDIF.

ENDFORM. "LE_ARQUIVO

*&---------------------------------------------------------------------*
*&      Form  EXECUTA - EXECUTA O BATCH INPUT
*&---------------------------------------------------------------------*

FORM executa.
*Tenta executar com Técnica Call Transaction
REFRESH t_msg.

CALL TRANSACTION 'V-03' USING t_bdc MODE p_modo UPDATE 'S'
       MESSAGES INTO t_msg.
  w_subrc = sy-subrc.
  CLEAR w_numerros.
  LOOP AT t_msg.
    IF t_msg-msgtyp NA 'IWS'.
      w_numerros = w_numerros + 1.
    ENDIF.
  ENDLOOP.

*Se transação terminou com erro, guarda dados em Pasta Batch Input
  IF w_subrc <> 0 OR w_numerros <> 0.

*Insere transação em Pasta Batch Input
    IF w_contabdc = 0.
      CALL FUNCTION 'BDC_OPEN_GROUP'
        EXPORTING
          client = sy-mandt
          group  = 'CLIENTES_09'
          keep   = 'X'
          user   = sy-uname.
    ENDIF.

    CALL FUNCTION 'BDC_INSERT'
      EXPORTING
        tcode     = 'V-03'
      TABLES
        dynprotab = t_bdc.
    w_contabdc  = w_contabdc + 1.
  ELSE.
    w_contapro = w_contapro + 1.
  ENDIF.

ENDFORM.

```
