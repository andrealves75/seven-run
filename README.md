import React, { useEffect, useState } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import { db } from '@/lib/firebase';
import { doc, getDoc, collection, query, where, getDocs, setDoc, updateDoc, addDoc, serverTimestamp, orderBy } from 'firebase/firestore';
import { UserProfile, Anamnesis, Evolution, WorkoutPlan } from '@/types';
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Textarea } from '@/components/ui/textarea';
import { 
  ArrowLeft, 
  User, 
  Activity, 
  History, 
  Plus, 
  Save,
  Weight,
  Ruler,
  Calendar,
  Heart,
  Dumbbell
} from 'lucide-react';
import { Avatar, AvatarFallback, AvatarImage } from '@/components/ui/avatar';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { 
  Dialog, 
  DialogContent, 
  DialogDescription, 
  DialogFooter, 
  DialogHeader, 
  DialogTitle, 
  DialogTrigger 
} from '@/components/ui/dialog';
import { toast } from 'sonner';

export default function StudentDetails() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();
  const { profile: coachProfile } = useAuth();
  const [student, setStudent] = useState<UserProfile | null>(null);
  const [anamnesis, setAnamnesis] = useState<Anamnesis | null>(null);
  const [evolutions, setEvolutions] = useState<Evolution[]>([]);
  const [plans, setPlans] = useState<WorkoutPlan[]>([]);
  const [loading, setLoading] = useState(true);
  const [isEditingAnamnesis, setIsEditingAnamnesis] = useState(false);
  const [anamnesisForm, setAnamnesisForm] = useState<Partial<Anamnesis>>({});

  // Plan creation state
  const [isAddPlanModalOpen, setIsAddPlanModalOpen] = useState(false);
  const [newPlan, setNewPlan] = useState({
    title: '',
    startDate: new Date().toISOString().split('T')[0],
    endDate: '',
  });

  const fetchData = async () => {
    if (!id) return;
    setLoading(true);
    try {
      // Fetch student profile
      const studentDoc = await getDoc(doc(db, 'users', id));
      if (studentDoc.exists()) {
        setStudent(studentDoc.data() as UserProfile);
      }

      // Fetch anamnesis
      const anamnesisDoc = await getDoc(doc(db, 'anamnesis', id));
      if (anamnesisDoc.exists()) {
        const data = anamnesisDoc.data() as Anamnesis;
        setAnamnesis(data);
        setAnamnesisForm(data);
      } else {
        setAnamnesisForm({ studentId: id });
      }

      // Fetch evolutions
      const evolutionsQ = query(collection(db, 'evolution'), where('studentId', '==', id), orderBy('date', 'desc'));
      const evolutionsSnapshot = await getDocs(evolutionsQ);
      setEvolutions(evolutionsSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() } as Evolution)));

      // Fetch plans
      const plansQ = query(collection(db, 'workoutPlans'), where('studentId', '==', id), orderBy('startDate', 'desc'));
      const plansSnapshot = await getDocs(plansQ);
      setPlans(plansSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() } as WorkoutPlan)));

    } catch (error) {
      console.error('Error fetching student details:', error);
      toast.error('Erro ao carregar dados do aluno');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, [id]);

  const handleSaveAnamnesis = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!id) return;

    try {
      const data = {
        ...anamnesisForm,
        studentId: id,
        updatedAt: new Date().toISOString(),
      };
      await setDoc(doc(db, 'anamnesis', id), data);
      setAnamnesis(data as Anamnesis);
      setIsEditingAnamnesis(false);
      toast.success('Anamnese salva com sucesso!');
    } catch (error) {
      console.error('Error saving anamnesis:', error);
      toast.error('Erro ao salvar anamnese');
    }
  };

  const handleCreatePlan = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!id || !coachProfile) return;

    try {
      const planData = {
        ...newPlan,
        studentId: id,
        coachId: coachProfile.uid,
        createdAt: new Date().toISOString(),
      };
      await addDoc(collection(db, 'workoutPlans'), planData);
      
      toast.success('Planilha criada com sucesso!');
      setIsAddPlanModalOpen(false);
      setNewPlan({
        title: '',
        startDate: new Date().toISOString().split('T')[0],
        endDate: '',
      });
      fetchData();
    } catch (error) {
      console.error('Error creating plan:', error);
      toast.error('Erro ao criar planilha');
    }
  };

  if (loading) {
    return <div className="flex items-center justify-center h-64 text-muted-foreground">Carregando...</div>;
  }

  if (!student) {
    return (
      <div className="text-center py-20">
        <p className="text-muted-foreground mb-4">Aluno não encontrado.</p>
        <Button onClick={() => navigate('/students')}>Voltar para lista</Button>
      </div>
    );
  }

  return (
    <div className="space-y-6">
      <header className="flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div className="flex items-center space-x-4">
          <Button variant="ghost" size="icon" onClick={() => navigate('/students')} className="text-muted-foreground hover:text-foreground">
            <ArrowLeft size={20} />
          </Button>
          <div className="flex items-center space-x-4">
            <Avatar className="h-12 w-12 border-2 border-primary">
              <AvatarImage src={student.photoURL || ''} />
              <AvatarFallback className="bg-secondary text-muted-foreground font-bold">
                {student.displayName.charAt(0)}
              </AvatarFallback>
            </Avatar>
            <div>
              <h1 className="text-2xl font-bold text-foreground tracking-tight">{student.displayName}</h1>
              <p className="text-sm text-muted-foreground">{student.email}</p>
            </div>
          </div>
        </div>
        
        <div className="flex items-center space-x-2">
          <Button variant="outline" className="border-border">Gerar Senha</Button>
          <Button className="bg-primary text-white font-bold">Enviar Mensagem</Button>
        </div>
      </header>

      <Tabs defaultValue="overview" className="space-y-6">
        <TabsList className="bg-secondary border border-border p-1 rounded-lg w-full md:w-auto overflow-x-auto flex-nowrap">
          <TabsTrigger value="overview" className="data-[state=active]:bg-primary data-[state=active]:text-white px-6">Resumo</TabsTrigger>
          <TabsTrigger value="anamnesis" className="data-[state=active]:bg-primary data-[state=active]:text-white px-6">Anamnese</TabsTrigger>
          <TabsTrigger value="evolution" className="data-[state=active]:bg-primary data-[state=active]:text-white px-6">Evolução</TabsTrigger>
          <TabsTrigger value="workouts" className="data-[state=active]:bg-primary data-[state=active]:text-white px-6">Planilhas</TabsTrigger>
        </TabsList>

        <TabsContent value="overview" className="space-y-6">
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            <Card className="bg-secondary/30 border-border p-4 rounded-xl flex items-center space-x-4">
              <div className="w-10 h-10 bg-primary/20 rounded-lg flex items-center justify-center text-primary">
                <Weight size={20} />
              </div>
              <div>
                <p className="text-xs text-muted-foreground uppercase font-bold tracking-wider">Peso</p>
                <p className="text-lg font-bold text-foreground">{anamnesis?.weight ? `${anamnesis.weight} kg` : '--'}</p>
              </div>
            </Card>
            <Card className="bg-secondary/30 border-border p-4 rounded-xl flex items-center space-x-4">
              <div className="w-10 h-10 bg-accent/20 rounded-lg flex items-center justify-center text-accent">
                <Ruler size={20} />
              </div>
              <div>
                <p className="text-xs text-muted-foreground uppercase font-bold tracking-wider">Altura</p>
                <p className="text-lg font-bold text-foreground">{anamnesis?.height ? `${anamnesis.height} cm` : '--'}</p>
              </div>
            </Card>
            <Card className="bg-secondary/30 border-border p-4 rounded-xl flex items-center space-x-4">
              <div className="w-10 h-10 bg-green-500/20 rounded-lg flex items-center justify-center text-green-500">
                <Activity size={20} />
              </div>
              <div>
                <p className="text-xs text-muted-foreground uppercase font-bold tracking-wider">Nível</p>
                <p className="text-lg font-bold text-foreground uppercase text-xs">{anamnesis?.experienceLevel || '--'}</p>
              </div>
            </Card>
          </div>

          <Card className="bg-card border-border rounded-xl overflow-hidden">
            <CardHeader className="border-b border-border bg-secondary/20">
              <CardTitle className="text-lg">Atividade Recente</CardTitle>
            </CardHeader>
            <CardContent className="p-0">
              <div className="divide-y divide-border">
                {evolutions.slice(0, 3).map(evo => (
                  <div key={evo.id} className="p-4 flex items-center justify-between hover:bg-secondary/10 transition-colors">
                    <div className="flex items-center space-x-4">
                      <div className="w-10 h-10 bg-secondary rounded-lg flex flex-col items-center justify-center text-[10px] font-bold">
                        <span className="text-primary">{new Date(evo.date).getDate()}</span>
                        <span className="text-muted-foreground uppercase">{new Date(evo.date).toLocaleString('pt-BR', { month: 'short' })}</span>
                      </div>
                      <div>
                        <p className="text-sm font-bold text-foreground">Registro de Evolução</p>
                        <p className="text-xs text-muted-foreground">{evo.weight}kg | {evo.notes || 'Sem notas'}</p>
                      </div>
                    </div>
                    <ChevronRight size={16} className="text-muted-foreground" />
                  </div>
                ))}
                {evolutions.length === 0 && (
                  <div className="p-8 text-center text-muted-foreground text-sm">Nenhuma atividade registrada.</div>
                )}
              </div>
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="anamnesis">
          <Card className="bg-card border-border rounded-xl">
            <CardHeader className="flex flex-row items-center justify-between border-b border-border mb-6">
              <div>
                <CardTitle>Anamnese Completa</CardTitle>
                <CardDescription>Histórico de saúde e objetivos do atleta.</CardDescription>
              </div>
              {!isEditingAnamnesis && (
                <Button variant="outline" size="sm" onClick={() => setIsEditingAnamnesis(true)} className="border-border">
                  Editar
                </Button>
              )}
            </CardHeader>
            <CardContent>
              {isEditingAnamnesis ? (
                <form onSubmit={handleSaveAnamnesis} className="space-y-6">
                  <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <div className="space-y-2">
                      <Label htmlFor="age">Idade</Label>
                      <Input 
                        id="age" 
                        type="number" 
                        className="bg-secondary border-border"
                        value={anamnesisForm.age || ''}
                        onChange={e => setAnamnesisForm({...anamnesisForm, age: parseInt(e.target.value)})}
                      />
                    </div>
                    <div className="space-y-2">
                      <Label htmlFor="weight">Peso (kg)</Label>
                      <Input 
                        id="weight" 
                        type="number" 
                        step="0.1"
                        className="bg-secondary border-border"
                        value={anamnesisForm.weight || ''}
                        onChange={e => setAnamnesisForm({...anamnesisForm, weight: parseFloat(e.target.value)})}
                      />
                    </div>
                    <div className="space-y-2">
                      <Label htmlFor="height">Altura (cm)</Label>
                      <Input 
                        id="height" 
                        type="number" 
                        className="bg-secondary border-border"
                        value={anamnesisForm.height || ''}
                        onChange={e => setAnamnesisForm({...anamnesisForm, height: parseInt(e.target.value)})}
                      />
                    </div>
                  </div>

                  <div className="space-y-2">
                    <Label htmlFor="level">Nível de Experiência</Label>
                    <select 
                      id="level"
                      className="w-full h-10 rounded-md bg-secondary border border-border px-3 text-sm focus:ring-2 focus:ring-primary outline-none"
                      value={anamnesisForm.experienceLevel || ''}
                      onChange={e => setAnamnesisForm({...anamnesisForm, experienceLevel: e.target.value as any})}
                    >
                      <option value="">Selecione...</option>
                      <option value="beginner">Iniciante</option>
                      <option value="intermediate">Intermediário</option>
                      <option value="advanced">Avançado</option>
                    </select>
                  </div>

                  <div className="space-y-2">
                    <Label htmlFor="conditions">Condições Médicas / Lesões</Label>
                    <Textarea 
                      id="conditions" 
                      placeholder="Descreva lesões prévias ou condições médicas..."
                      className="bg-secondary border-border min-h-[100px]"
                      value={anamnesisForm.medicalConditions || ''}
                      onChange={e => setAnamnesisForm({...anamnesisForm, medicalConditions: e.target.value})}
                    />
                  </div>

                  <div className="space-y-2">
                    <Label htmlFor="goals">Objetivos</Label>
                    <Textarea 
                      id="goals" 
                      placeholder="Ex: Correr minha primeira maratona, baixar tempo nos 5km..."
                      className="bg-secondary border-border min-h-[100px]"
                      value={anamnesisForm.goals || ''}
                      onChange={e => setAnamnesisForm({...anamnesisForm, goals: e.target.value})}
                    />
                  </div>

                  <div className="flex justify-end space-x-3 pt-4">
                    <Button type="button" variant="ghost" onClick={() => setIsEditingAnamnesis(false)}>Cancelar</Button>
                    <Button type="submit" className="bg-primary text-white font-bold">
                      <Save size={18} className="mr-2" />
                      Salvar Alterações
                    </Button>
                  </div>
                </form>
              ) : (
                <div className="space-y-8">
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                    <div className="space-y-4">
                      <div>
                        <h4 className="text-xs font-bold uppercase tracking-widest text-primary mb-2">Dados Biométricos</h4>
                        <div className="grid grid-cols-2 gap-4">
                          <div className="bg-secondary/30 p-3 rounded-lg border border-border/50">
                            <p className="text-[10px] text-muted-foreground uppercase">Idade</p>
                            <p className="font-bold">{anamnesis?.age || '--'} anos</p>
                          </div>
                          <div className="bg-secondary/30 p-3 rounded-lg border border-border/50">
                            <p className="text-[10px] text-muted-foreground uppercase">Nível</p>
                            <p className="font-bold uppercase text-xs">{anamnesis?.experienceLevel || '--'}</p>
                          </div>
                        </div>
                      </div>
                      
                      <div>
                        <h4 className="text-xs font-bold uppercase tracking-widest text-primary mb-2">Condições Médicas</h4>
                        <p className="text-sm text-foreground bg-secondary/30 p-4 rounded-lg border border-border/50 min-h-[80px]">
                          {anamnesis?.medicalConditions || 'Nenhuma informação registrada.'}
                        </p>
                      </div>
                    </div>

                    <div className="space-y-4">
                      <div>
                        <h4 className="text-xs font-bold uppercase tracking-widest text-primary mb-2">Objetivos do Atleta</h4>
                        <p className="text-sm text-foreground bg-secondary/30 p-4 rounded-lg border border-border/50 min-h-[160px]">
                          {anamnesis?.goals || 'Nenhum objetivo registrado.'}
                        </p>
                      </div>
                    </div>
                  </div>
                  
                  <div className="pt-4 text-[10px] text-muted-foreground italic">
                    Última atualização: {anamnesis?.updatedAt ? new Date(anamnesis.updatedAt).toLocaleString('pt-BR') : 'Nunca'}
                  </div>
                </div>
              )}
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="evolution">
          <Card className="bg-card border-border rounded-xl">
            <CardHeader className="flex flex-row items-center justify-between border-b border-border mb-6">
              <div>
                <CardTitle>Histórico de Evolução</CardTitle>
                <CardDescription>Acompanhamento de métricas e fotos.</CardDescription>
              </div>
              <Button size="sm" className="bg-primary text-white font-bold">
                <Plus size={18} className="mr-2" />
                Novo Registro
              </Button>
            </CardHeader>
            <CardContent>
              <div className="space-y-4">
                {evolutions.length > 0 ? (
                  evolutions.map(evo => (
                    <div key={evo.id} className="p-4 rounded-xl bg-secondary/30 border border-border/50 flex flex-col md:flex-row gap-4">
                      {evo.photoURL && (
                        <div className="w-full md:w-32 h-32 rounded-lg overflow-hidden bg-black flex-shrink-0">
                          <img src={evo.photoURL} alt="Evolução" className="w-full h-full object-cover" />
                        </div>
                      )}
                      <div className="flex-1 space-y-2">
                        <div className="flex items-center justify-between">
                          <div className="flex items-center space-x-2 text-primary">
                            <Calendar size={14} />
                            <span className="text-xs font-bold">{new Date(evo.date).toLocaleDateString('pt-BR')}</span>
                          </div>
                          <div className="flex space-x-4">
                            {evo.weight && (
                              <div className="text-center">
                                <p className="text-[10px] text-muted-foreground uppercase">Peso</p>
                                <p className="text-sm font-bold">{evo.weight}kg</p>
                              </div>
                            )}
                            {evo.vo2Max && (
                              <div className="text-center">
                                <p className="text-[10px] text-muted-foreground uppercase">VO2 Max</p>
                                <p className="text-sm font-bold text-accent">{evo.vo2Max}</p>
                              </div>
                            )}
                          </div>
                        </div>
                        <p className="text-sm text-muted-foreground italic">"{evo.notes || 'Sem observações.'}"</p>
                      </div>
                    </div>
                  ))
                ) : (
                  <div className="text-center py-10">
                    <History className="mx-auto text-muted-foreground mb-4 opacity-20" size={48} />
                    <p className="text-muted-foreground">Nenhum registro de evolução ainda.</p>
                  </div>
                )}
              </div>
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="workouts">
          <Card className="bg-card border-border rounded-xl">
            <CardHeader className="flex flex-row items-center justify-between border-b border-border mb-6">
              <div>
                <CardTitle>Planilhas de Treino</CardTitle>
                <CardDescription>Gerencie as planilhas do atleta.</CardDescription>
              </div>
              
              <Dialog open={isAddPlanModalOpen} onOpenChange={setIsAddPlanModalOpen}>
                <DialogTrigger asChild>
                  <Button size="sm" className="bg-primary text-white font-bold">
                    <Plus size={18} className="mr-2" />
                    Nova Planilha
                  </Button>
                </DialogTrigger>
                <DialogContent className="bg-card border-border text-foreground">
                  <DialogHeader>
                    <DialogTitle>Criar Nova Planilha</DialogTitle>
                    <DialogDescription className="text-muted-foreground">
                      Defina o título e o período da nova planilha de treinos.
                    </DialogDescription>
                  </DialogHeader>
                  <form onSubmit={handleCreatePlan} className="space-y-4 py-4">
                    <div className="space-y-2">
                      <Label htmlFor="planTitle">Título da Planilha</Label>
                      <Input 
                        id="planTitle" 
                        placeholder="Ex: Ciclo de Base - 12 Semanas" 
                        className="bg-secondary border-border"
                        value={newPlan.title}
                        onChange={e => setNewPlan({...newPlan, title: e.target.value})}
                        required
                      />
                    </div>
                    <div className="grid grid-cols-2 gap-4">
                      <div className="space-y-2">
                        <Label htmlFor="startDate">Data de Início</Label>
                        <Input 
                          id="startDate" 
                          type="date" 
                          className="bg-secondary border-border"
                          value={newPlan.startDate}
                          onChange={e => setNewPlan({...newPlan, startDate: e.target.value})}
                          required
                        />
                      </div>
                      <div className="space-y-2">
                        <Label htmlFor="endDate">Data de Término</Label>
                        <Input 
                          id="endDate" 
                          type="date" 
                          className="bg-secondary border-border"
                          value={newPlan.endDate}
                          onChange={e => setNewPlan({...newPlan, endDate: e.target.value})}
                          required
                        />
                      </div>
                    </div>
                    <DialogFooter>
                      <Button type="button" variant="ghost" onClick={() => setIsAddPlanModalOpen(false)}>Cancelar</Button>
                      <Button type="submit" className="bg-primary text-white font-bold">Criar Planilha</Button>
                    </DialogFooter>
                  </form>
                </DialogContent>
              </Dialog>
            </CardHeader>
            <CardContent>
              <div className="space-y-4">
                {plans.length > 0 ? (
                  plans.map(plan => (
                    <div key={plan.id} className="p-4 rounded-xl bg-secondary/30 border border-border/50 flex items-center justify-between group hover:border-primary/50 transition-colors">
                      <div className="flex items-center space-x-4">
                        <div className="w-10 h-10 bg-primary/20 rounded-lg flex items-center justify-center text-primary">
                          <Calendar size={20} />
                        </div>
                        <div>
                          <p className="font-bold text-foreground">{plan.title}</p>
                          <p className="text-xs text-muted-foreground">
                            {new Date(plan.startDate).toLocaleDateString('pt-BR')} - {new Date(plan.endDate).toLocaleDateString('pt-BR')}
                          </p>
                        </div>
                      </div>
                      <Button variant="ghost" size="sm" className="text-accent hover:text-accent hover:bg-accent/10">
                        Gerenciar Treinos
                        <ChevronRight size={16} className="ml-2" />
                      </Button>
                    </div>
                  ))
                ) : (
                  <div className="text-center py-10">
                    <Dumbbell className="mx-auto text-muted-foreground mb-4 opacity-20" size={48} />
                    <p className="text-muted-foreground">Nenhuma planilha ativa encontrada.</p>
                    <Button variant="outline" className="mt-4 border-border" onClick={() => setIsAddPlanModalOpen(true)}>Criar Primeira Planilha</Button>
                  </div>
                )}
              </div>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}

function ChevronRight(props: any) {
  return (
    <svg
      {...props}
      xmlns="http://www.w3.org/2000/svg"
      width="24"
      height="24"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth="2"
      strokeLinecap="round"
      strokeLinejoin="round"
    >
      <path d="m9 18 6-6-6-6" />
    </svg>
  );
}
