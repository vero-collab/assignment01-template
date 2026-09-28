#include 
#include 
#include 

#define MAX_SCHEDULES 300
#define MAX_STOPS 200
#define MAX_LINE 256
#define MAX_RECS 2000

typedef struct {
    char id[16], dir;
    int n_stops, stop_ids[MAX_STOPS];
    char stop_names[MAX_STOPS][128];
} Sched;

typedef struct {
    int m, d, y, hr, min, total_sec, stop_id;
    char type, iso[32];
} Rec;

Sched scheds[MAX_SCHEDULES];
int n_scheds = 0;

const char* get_stop_name(int id) {
    for (int i = 0; i < n_scheds; i++)
        for (int j = 0; j < scheds[i].n_stops; j++)
            if (scheds[i].stop_ids[j] == id) return scheds[i].stop_names[j];
    return NULL;
}

int load_schedules(const char *list_file) {
    FILE *fp = fopen(list_file, "r");
    if (!fp) return 0;

    char line[MAX_LINE];
    while (fgets(line, sizeof(line), fp)) {
        line[strcspn(line, "\r\n")] = 0;
        if (!line[0]) continue;

        FILE *sfp = fopen(line, "r");
        if (!sfp) continue;

        Sched *s = &scheds[n_scheds];
        char *base = strrchr(line, '/');
        base = base ? base + 1 : line;

        // Parse route ID and direction from path (e.g. route_101_EB.txt)
        char temp[128];
        strcpy(temp, base);
        char *dot = strrchr(temp, '.'); if (dot) *dot = 0;
        char *underscore = strrchr(temp, '_');
        s->dir = underscore ? underscore[1] : '?';
        if (underscore) *underscore = 0;
        char *first_u = strchr(temp, '_');
        strcpy(s->id, first_u ? first_u + 1 : temp);

        if (fgets(line, sizeof(line), sfp))
            sscanf(line, "%*s %*s %d", &s->n_stops);

        int idx = 0;
        while (fgets(line, sizeof(line), sfp) && idx < MAX_STOPS) {
            line[strcspn(line, "\r\n")] = 0;
            char *comma = strchr(line, ',');
            if (comma) {
                *comma = 0;
                s->stop_ids[idx] = atoi(line);
                strcpy(s->stop_names[idx++], comma + 1);
            }
        }
        s->n_stops = idx;
        fclose(sfp);
        n_scheds++;
    }
    fclose(fp);
    return 1;
}

int main(int argc, char *argv[]) {
    if (argc < 2 || !load_schedules(argv[1])) return 1;

    int n_recs = 0;
    if (scanf("%d\n", &n_recs) != 1) {
        // Milestone mode: simple summary printout
        printf("Processing %d schedule files...\n", n_scheds);
        for (int i = 0; i < n_scheds; i++)
            printf("schedule #%d is route %s it has %d stops and is in the %c direction\n",
                   i + 1, scheds[i].id, scheds[i].n_stops, scheds[i].dir);
        return 0;
    }

    Rec recs[MAX_RECS];
    char line[MAX_LINE];
    int count = 0;

    while (count < n_recs && fgets(line, sizeof(line), stdin)) {
        line[strcspn(line, "\r\n")] = 0;
        if (!line[0]) continue;

        char d_str[32], t_str[32], type_str[8], loc[32] = "";
        if (sscanf(line, "%31[^,],%31[^,],%7[^,],%31s", d_str, t_str, type_str, loc) < 3) {
            printf("Invalid data format\n"); return 0;
        }

        Rec *r = &recs[count];
        r->type = type_str[0];

        char ampm[8];
        sscanf(d_str, "%d-%d-%d", &r->m, &r->d, &r->y);
        sscanf(t_str, "%d:%d %2s", &r->hr, &r->min, ampm);
        if ((ampm[0] == 'P' || ampm[0] == 'p') && r->hr < 12) r->hr += 12;
        if ((ampm[0] == 'A' || ampm[0] == 'a') && r->hr == 12) r->hr = 0;

        snprintf(r->iso, sizeof(r->iso), "%04d-%02d-%02dT%02d:%02d:00", r->y, r->m, r->d, r->hr, r->min);
        r->total_sec = ((r->y * 365 + r->m * 30 + r->d) * 24 + r->hr) * 3600 + r->min * 60;

        if (r->type != 'M') {
            r->stop_id = atoi(loc);
            if (!get_stop_name(r->stop_id)) { printf("Invalid stop ID\n"); return 0; }
        } else r->stop_id = -1;

        count++;
    }

    // Reconstruction loop matching exit -> entry
    for (int i = 0; i < count; i++) {
        if (recs[i].type == 'X' || recs[i].type == 'M') {
            int entry_idx = -1;
            for (int j = i + 1; j < count; j++) {
                if (recs[j].type == 'E') { entry_idx = j; break; }
            }
            if (entry_idx == -1) continue;

            int ex_stop = recs[i].stop_id, guessed = 0;
            if (recs[i].type == 'M') {
                guessed = 1;
                int freq[10000] = {0}, max_f = 0;
                for (int k = 0; k < count; k++) {
                    if (recs[k].type == 'E' && recs[k].stop_id == recs[entry_idx].stop_id && recs[k].hr == recs[entry_idx].hr) {
                        if (k > 0 && recs[k-1].type == 'X') {
                            freq[recs[k-1].stop_id]++;
                            if (freq[recs[k-1].stop_id] > max_f) max_f = freq[recs[k-1].stop_id];
                        }
                    }
                }
                for (int s = 0; s < 10000 && max_f > 0; s++) {
                    if (freq[s] == max_f) { ex_stop = s; break; }
                }
            }

            int matches[MAX_SCHEDULES], n_matches = 0;
            for (int s = 0; s < n_scheds; s++) {
                int has_e = 0, has_x = 0;
                for (int st = 0; st < scheds[s].n_stops; st++) {
                    if (scheds[s].stop_ids[st] == recs[entry_idx].stop_id) has_e = 1;
                    if (scheds[s].stop_ids[st] == ex_stop) has_x = 1;
                }
                if (has_e && has_x) matches[n_matches++] = s;
            }

            int sel = 0;
            if (n_matches > 1) {
                printf("Ambiguous trip detected:\nEntry: %d (%s)\nExit: %d (%s)\nPossible routes:\n",
                       recs[entry_idx].stop_id, get_stop_name(recs[entry_idx].stop_id), ex_stop, get_stop_name(ex_stop));
                for (int m = 0; m < n_matches; m++)
                    printf("%d. %s %c\n", m + 1, scheds[matches[m]].id, scheds[matches[m]].dir);
                printf("Select route (1-%d): ", n_matches);
                if (scanf("%d", &sel) != 1) sel = 1; else sel--;
                sel = matches[sel];
            } else sel = matches[0];

            int dur = recs[i].total_sec - recs[entry_idx].total_sec;
            printf("%s %c,%s,%s,%s,%s,%d:%02d:%02d%s\n",
                   scheds[sel].id, scheds[sel].dir,
                   get_stop_name(recs[entry_idx].stop_id), get_stop_name(ex_stop),
                   recs[entry_idx].iso, recs[i].iso,
                   dur / 3600, (dur % 3600) / 60, dur % 60,
                   guessed ? ",Guessed" : "");

            i = entry_idx;
        }
    }
    return 0;
}
