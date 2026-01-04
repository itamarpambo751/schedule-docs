# 1. Apresentação do Problema e da Ideia do Sistema

## 1.1 O problema que o sistema resolve

Em qualquer unidade de saúde (hospital, clínica ou consultório):

- Há múltiplos profissionais, cada um com horários diferentes.
- Consultas podem ser presenciais, online (telemedicina) ou domiciliares.
- Pacientes têm idades, gêneros, restrições de plano de saúde e preferências.
- Algumas datas ou horários não podem ser usados (feriados, reuniões, folgas).

Se cada consulta fosse marcada **manual ou isoladamente**, surgiriam problemas sérios:

- **Conflitos de horário**: dois pacientes marcados para o mesmo médico ao mesmo tempo.
- **Ineficiência**: agendas subutilizadas ou horários vagos desperdiçados.
- **Erros humanos**: esquecimento de folgas, feriados ou horários de almoço.
- **Dificuldade de previsão**: não se sabe quantos atendimentos vão ocorrer em cada dia.

O sistema existe para **resolver essas dores de forma automatizada e confiável**.

---

## 1.2 A ideia central do sistema

O sistema não trata cada consulta individualmente antes de existir um horário. Ele trabalha assim:

1. **Agenda**  
   Define as **regras gerais** do atendimento: dias da semana, horários, duração das consultas, tipo de serviço, categoria da especialidade e limitações (gênero, idade, etc.).

2. **Slot (dia de atendimento)**  
   Cada dia que se enquadra na agenda se torna um **slot**, que representa **todo o conjunto de horários possíveis daquele dia**, e não cada hora individualmente. O slot é o “container” que vai receber **horários específicos (HourBox)** mais tarde.

3. **HourBox (horário real dentro do slot)**  
   Com base na agenda, na duração das consultas e nas regras de capacidade, o sistema gera **horários disponíveis para marcação** dentro de cada slot.  
   Esses horários são o que o paciente verá para marcar consulta.

4. **Recorrência**  
   Permite que a agenda se repita no tempo, gerando slots automaticamente em dias futuros, sem precisar criar manualmente cada dia.

5. **Exclusões**  
   Permite remover dias ou horários específicos da agenda (feriados, folgas, reuniões), mesmo que a regra geral diga que deveria haver atendimento.

6. **Horários de trabalho dos profissionais**  
   Cada profissional define quando está disponível. O sistema só atribui profissionais aos horários de um slot se eles estiverem realmente trabalhando naquele dia e horário.

7. **Marcação de consultas**  
   Só depois que os slots e horários são gerados é que o paciente consegue marcar. Isso garante que não haja conflitos e que todas as regras sejam respeitadas automaticamente.

---

## 1.3 Por que isso importa

- **Agilidade**: evita a necessidade de conferências manuais de agendas.
- **Segurança**: aplica todas as regras automaticamente (idade, gênero, folgas, feriados).
- **Escalabilidade**: permite gerenciar dezenas de profissionais e centenas de pacientes sem erro.
- **Transparência**: cada passo (geração de slot, atribuição de profissional, exclusão, marcação) é registrado e auditável.

